---
title: SGX SDK相关
tags:
  - SGX
  - 系统安全
date: 2016-11-24 11:40:00
---

### Enclave Definition Language
EDL文件，用来描述enclave当中的trusted和untrusted部分。在Linux中，Edger8r Tool通过这个文件来创建enclave的C wrapper函数，也即ECALL和OCALL所使用的函数。 

	//EDL Template
	enclave	{
		//包含文件和作为参数的数据结构
		trusted {
			//任何enclave_t.h中包含的文件
			//trusted function原型
		};
	
		untrusted {
			//任何enclave_u.t中包含的文件
			//untrusted function原型
		};
	};
	
EDL文件不允许include有自定义类型的头文件。对于全局的包含文件，也不会包含在enclave当中。这类情况会使用不同的头文件，例如SGX会利用SDK所提供的stdio.h，而应用会使用由编译器提供的stdio.h。  

#### EDL中的数据
EDL中可以使用基本的关键字，包括

	char, short, long, int, float, double, void, int8_t, int16_t, int32_t, 	int64_t, size_t, wchar_t, uint8_t, uint16_t, uint32_t, uint64_t, unsigned, 	struct, enum, union
	
其它类型可以在头文件中包含。用户定义的数据类型可以在EDL中使用，但要遵守其编写的规范。正确的定义如下

		enclave{		include "user_types.h" 
				struct struct_foo_t { 			uint32_t struct_foo_0; 
			uint64_t struct_foo_1;		};
					enum enum_foo_t { 
			ENUM_FOO_0 = 0,			ENUM_FOO_1 = 1 };		}; 
			trusted {					public void test_char(char val); public void test_int(int val);			public void test_long(long long val);
	};

#### EDL中的指针
EDL中定义有一些和指针一起使用的值，这些值是用在ECALL和OCALL时使用的参数上的。指针需要用in,out,或者user_check进行明确的修饰。其中[in]和[out]说明的是方向：  

- [in]表示参数从调用方传递到被调用放，对ECALL来说，in是从应用程序传递到enclave中，对OCALL来说则表示参数从应用程序传递到enclave中。  
- [out]表示参数是从被调用方返回到调用方。对ECALL来说，out表示参数从enclave传递到应用中，对OCALL来说则是从应用传递给enclave。  
- [in]和[out]组合使用表示参数是双向传递的。  

方向属性能够用来提供保护，但会降低性能。如果使用user_check，则表示在不可信内存中的数据会在使用前进行验证。但[in]和[out]不支持包含有指针的结构体，这种情况必须使用user_check，并进行手动的验证。为了保证copy指针指向数据的安全性，它们还会和size，count，sizefun等一起使用。  

#### 其它数据类型
string和wstring表示参数是一个以NULL结束的字符串。  

EDL支持用户定义的数据类型，但是不能定义在头文件中。任何使用typedef的基本类型，也都是用户定义的数据类型。有一些数据类型必须指定EDL属性，例如isptr，isary，readonly等，否则edger8r在编译时会报错。  

propagate_error是OCALL的一个属性，如果使用这个属性，则enclave中的errno属性，会在OCALL返回之前被覆写为untrusted域中的errno中的值。在OCALL完成之后，无论OCALL是否成功，trusted域中的都会在OCALL完成之后更新。如果function失败了，那么errno就会检查是否出错，而如果函数成功了，那么OCALL就被允许修改errno的值。

#### ECALL访问/配置
默认的情况下，ECALL函数是不能直接被任何untrusted functions调用的。为了允许应用程序直接调用一个ECALL函数，则这个ECALL必须用public关键字来修饰。  
Enclave配置文件是一个XML文件，它包含了用户定义的enclave参数，也是enclave项目的一部分。sgx_sign利用这个文件作为输入，来创建enclave的signature和metadata，它包括有这些项：

	<EnclaveConfiguration>	<ProdID>100</ProdID> 	<ISVSVN>1</ISVSVN> 
	<StackMaxSize>0x50000</StackMaxSize>
	<HeapMaxSize>0x100000</HeapMaxSize>
	<TCSNum>1</TCSNum>
	<TCSPolicy>1</TCSPolicy>
	<DisableDebug>0</DisableDebug>
	<MiscSelect>0</MiscSelect>
	<MiscMask>0xFFFFFFFF</MiscMask>	</EnclaveConfiguration>

#### Enclave加载
Enclave的源代码被编译为一个共享对象，为了使用一个enclave，enclave.so应该通过调用sgx_create_enclave()函数来加载到受保护的内存中。在第一次加载一个enclave时，加载器会获取launch token并且将它保存到token参数当中，用户能够将它保存在一个文件中，并且在之后加载时，从文件中获取token。而卸载enclave则是由用户调用sgx_destory_enclave(sgx_enclave_id_t)来实现的。  

	#define ENCLAVE_FILE _T("Enclave.signed.so")	sgx_enclave_id_t eid;	sgx_status_t ret = SGX_SUCCESS; 
	sgx_launch_token_t token = {0};	int updated = 0;
	//创建
	sgx_create_enclave(ENCLAVE_FILE, SGX_DEBUG_FLAG, &token, &updated, &eid, NULL);
	//摧毁
	sgx_destroy_enclave(eid);
	
#### Untrusted/Trusted Library Functions
untrusted函数只能在应用中，也就是enclave的外部调用。这些函数包括：

- Enclave的创建和摧毁
- Quoting（用来确定处于SGX环境中）
- untrusted key交换
- 平台服务和启动控制  

trusted库和enclave binary静态链接，它们只能在enclave内部使用。
Trusted Runtime System是SDK的一个关键组件，它提供enclave的入口逻辑，其他的helper函数，以及自定义的异常处理。  
Trusted Service Library对数据进行保护，它包括有：

－ Wrapper函数
－ sealing/unsealing
－ Trusted平台服务函数

#### SDK
SGX的SDK目前提供了两种模式，hardware模式，也即在物理上拥有sgx的计算机上能够使用；而simulation则是模拟拥有sgx的计算机。在DEBUG模式中，开发者能够直接使用被签名过的enclave.signed.so，而不需要自己去进行签名。
	
	