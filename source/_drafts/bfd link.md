elf_x86_64_relocate_section -> _bfd_final_linke_relocate
address  
value  
addend  

bfd_vma relocation
通常relocation = value + addend;  
value是被修正的符号原有的值； 

rel->r_offset  为 address  入口偏移量  
value 为重定位符号的值，也即其位置  
addend是rel->r_addend 修改位置中的隐式加数  

relocation ＝ value + addend 重定位符号的值 ＋ 加数

relocation -= input_section->output_section->vma + input_section->output_offset
减去输出段到输入段的偏移

if(howto -> pcrel_offset)
 relocation -= address; 再减掉本来的偏移
 

 bfd_elf_final_link
 elf_link_input_bfd
 elf_backend_relocate_section
 elf_x86_64_relocate_section完成一个section的重定位。
 
 	for(;rel < relend; wrel++, rel++)//对每一个重定位项进行处理
 
 首先从htab中，根据rel->r_info来退货去符号的序号：
 
 	r_symndx = htab->r_sym(rel->r_info)
 	
 随后根据这个r_symndx的范围，判断它是本地还是全局的symbol，获取relocation信息；relocation是某一个重定位项的重定位信息，也就是它在output当中的，新的地址。记录的是符号对重定位位置的偏移，所以它们的位置应该是一致的。  
 
在外部的符号，确实需要通过链接器来搞定，因为这些符号可能不在插件中的set当中了；静态的函数是不会有符号表的。  
