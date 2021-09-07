---
title: "关闭内核内存写保护"
date: 2021-10-17T10:12:10+08:00
categories: ["操作系统"]
draft: false
---

## 内存写保护

某块内存的读写执行权限实际上由其页表项(PTE)标记.修改页表项内容可以更改其权限.

TODO: 曾经看到过一个项目:根据用户态传递的地址,在内核态查找到 PTE 并修改权限,用户态再给只读变量赋值.后面找到了再补上链接吧

内核中的内存实际上与用户态的内存没有本质上的区别,都可以通过修改页表项改变其权限.

## 内核设置只读数据段为只读

内核初始化函数 init/main.c/start_kernel 中.有这样的一层调用关系:
`start_kernel > arch_call_rest_init > rest_init > kernel_init > mark_readonly > mark_rodata_ro`.
其中 mark_rodata_ro 由各个架构分别实现.

## ARM64中内存设置只读权限

以 ARM64 为例,看看写保护是如何生效的.

在 arch/arm64/mm/mmu.c 中可以找到 mark_rodata_ro .其中 __pa_symbol 对地址进行常数 kimage_voffset 偏移,并且 kimage_voffset 是导出符号.

```c
void mark_rodata_ro(void)
{
	unsigned long section_size;

	/*
	 * mark .rodata as read only. Use __init_begin rather than __end_rodata
	 * to cover NOTES and EXCEPTION_TABLE.
	 */
	section_size = (unsigned long)__init_begin - (unsigned long)__start_rodata;
	update_mapping_prot(__pa_symbol(__start_rodata), (unsigned long)__start_rodata,
			    section_size, PAGE_KERNEL_RO);

	debug_checkwx();
}
```

 update_mapping_prot 4 个参数分别是:物理地址,虚拟地址,长度,权限.此时就能根据地址范围设置权限了.
此处的物理地址与虚拟地址之间有相对固定的偏移量,四个参数含义明确.
 __create_pgd_mapping 更新完地址后,通过 flush_tlb_kernel_range 刷新 TLB,
当时操作系统课上 [TLB](https://en.wikipedia.org/wiki/Translation_lookaside_buffer) 被称为"快表",
实际上它是内存的缓存,内存更新后要让缓存失效(或者重新加载)

需要注意 init_mm 符号没有导出,在内核模块中使用需要特殊处理,不属于这篇文章的范围,不做赘述.

```c
static void update_mapping_prot(phys_addr_t phys, unsigned long virt,
				phys_addr_t size, pgprot_t prot)
{
	if ((virt >= PAGE_END) && (virt < VMALLOC_START)) {
		pr_warn("BUG: not updating mapping for %pa at 0x%016lx - outside kernel range\n",
			&phys, virt);
		return;
	}

	__create_pgd_mapping(init_mm.pgd, phys, virt, size, prot, NULL,
			     NO_CONT_MAPPINGS);

	/* flush the TLBs after updating live kernel mappings */
	flush_tlb_kernel_range(virt, virt + size);
}
```

注意函数指针 pgtable_alloc 为空.
这个函数把先前的需要处理的内存的长度拆分成小块,并逐个处理.
由于入参是 pgd, 所以拆分后应该是 p4d, 后续还会拆分出 pud, pmd, pte.
这是 Linux 中目前使用的五级页表.

```c
static void __create_pgd_mapping(pgd_t *pgdir, phys_addr_t phys,
				 unsigned long virt, phys_addr_t size,
				 pgprot_t prot,
				 phys_addr_t (*pgtable_alloc)(int),
				 int flags)
{
	unsigned long addr, end, next;
	pgd_t *pgdp = pgd_offset_pgd(pgdir, virt);

	/*
	 * If the virtual and physical address don't have the same offset
	 * within a page, we cannot map the region as the caller expects.
	 */
	if (WARN_ON((phys ^ virt) & ~PAGE_MASK))
		return;

	phys &= PAGE_MASK;
	addr = virt & PAGE_MASK;
	end = PAGE_ALIGN(virt + size);

	do {
		next = pgd_addr_end(addr, end);
		alloc_init_pud(pgdp, addr, next, phys, prot, pgtable_alloc,
			       flags);
		phys += next - addr;
	} while (pgdp++, addr = next, addr != end);
}
```

TODO: 根据情况决定是否有必要继续补充多级页表中间部分内容

最终通过 set_pte 设置页表项

```c
static void init_pte(pmd_t *pmdp, unsigned long addr, unsigned long end,
		     phys_addr_t phys, pgprot_t prot)
{
	pte_t *ptep;

	ptep = pte_set_fixmap_offset(pmdp, addr);
	do {
		pte_t old_pte = READ_ONCE(*ptep);

		set_pte(ptep, pfn_pte(__phys_to_pfn(phys), prot));

		/*
		 * After the PTE entry has been populated once, we
		 * only allow updates to the permission attributes.
		 */
		BUG_ON(!pgattr_change_is_safe(pte_val(old_pte),
					      READ_ONCE(pte_val(*ptep))));

		phys += PAGE_SIZE;
	} while (ptep++, addr += PAGE_SIZE, addr != end);

	pte_clear_fixmap();
}
```

# 修改内存权限

在 init_pte 中可以看出,修改权限需要拿到 ptep 并对修改页表项.
同文件中的 kern_addr_valid 函数刚好可以拿到 ptep .
对其进行简单修改就可以实现开关写保护的功能.
 init_mm_ptr 是指向 init_mm 的指针,请自行想办法获取,本文不做描述.
修改完内存不要忘记刷新 tlb .~~虽然刷新整个tlb比较粗暴,但是它使用方式足够简单,只要不是频繁调用完全可以接受,比如我就可以接受~~

```c
#include <asm/pgtable-types.h>
#include <asm/pgtable.h>

static int get_kern_addr_ptep(unsigned long addr, pte_t **ptepp)
{
	pgd_t *pgdp;
	p4d_t *p4dp;
	pud_t *pudp, pud;
	pmd_t *pmdp, pmd;
	pte_t *ptep, pte;

	addr = arch_kasan_reset_tag(addr);
	if ((((long)addr) >> VA_BITS) != -1UL)
		return 0;

	pgdp = (init_mm_ptr->pgd +
		(((addr) >> PGDIR_SHIFT) & (PTRS_PER_PGD - 1)));
	if (pgd_none(READ_ONCE(*pgdp)))
		return 0;

	p4dp = p4d_offset(pgdp, addr);
	if (p4d_none(READ_ONCE(*p4dp)))
		return 0;

	pudp = pud_offset(p4dp, addr);
	pud = READ_ONCE(*pudp);
	if (pud_none(pud))
		return 0;

	if (pud_sect(pud))
		return pfn_valid(pud_pfn(pud));

	pmdp = pmd_offset(pudp, addr);
	pmd = READ_ONCE(*pmdp);
	if (pmd_none(pmd))
		return 0;

	if (pmd_sect(pmd))
		return pfn_valid(pmd_pfn(pmd));

	ptep = pte_offset_kernel(pmdp, addr);
	*ptepp = ptep;
	pte = *ptep;
	if (pte_none(pte))
		return 0;

	return pfn_valid(pte_pfn(pte));
}

void enable_wp(unsigned long addr)
{
	pte_t *ptep;
	if (!get_kern_addr_ptep(addr, &ptep))
		return;
	set_pte(ptep, pte_wrprotect(*ptep));
	flush_tlb_all();
}

void disable_wp(unsigned long addr)
{
	pte_t *ptep;
	if (!get_kern_addr_ptep(addr, &ptep))
		return;
	set_pte(ptep, pte_mkwrite(*ptep));
	flush_tlb_all();
}
```

完结撒花~

