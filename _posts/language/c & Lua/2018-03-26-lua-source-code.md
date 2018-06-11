---
layout: post
title: Lua 源码
category: 语言
tags: language
keywords: language lua c
description:
---


```c
// src/lvm.c
...
void get_type(const TValue *obj) {
    printf("%d ", ttype(obj));
    if (ttisnumber(obj)) {
       int data = obj->value_.n;
       printf("is number: %d", data);
    } else if (ttisstring(obj)) {
        printf("is string: ");
        TString *ts;
        ts = &(obj->value_.gc->ts);
        // ts = lua_State->CallInfo->u.LuaCall.(stack-base-for-this-call).Value.GCObject.TString
        // 从内存上看其实字符串的值直接放在了 TString 后面，这样还能省掉一个成员：(lstring.c 108)
        // 因此，下方代码实现了读取并打印 ts pointer 后面的 read_length 位的字符
        // ts 等于当前线程状态下 current function 的 callinfo
        int i;
        int s_length = 32;
        for(i = 0; i < s_length; i++){
          printf("%c ", ((char *)(ts-s_length))[i]);
        };
        for(i = 0; i < s_length; i++){
          printf("%c ", ((char *)(ts+1))[i]);
        };
    } else if (ttisnil(obj)) {
       printf("is nil");
    } else if (ttisshrstring(obj)) {
       printf("is shrstring");
    } else if (ttislngstring(obj)) {
       printf("is lngstring");
    } else if (ttistable(obj)) {
       printf("is table");
    } else if (ttisfunction(obj)) {
       printf("is function");
    } else if (ttisclosure(obj)) {
       printf("is closure");
    } else if (ttisLclosure(obj)) {
       printf("is Lclosure");
    } else {
       printf("type not found");
    };
    printf("\n");
}

void print_opcode_detail(int opcode, StkId base, TValue *k, CallInfo *ci, Instruction i) {
  // printf("%d", opcode);
  vmdispatch (opcode) {
    vmcase(OP_LOADK,
      // Register A (specified in instruction field A)
      TValue *register_A = RA(i);
      print_type(register_A);
      // Kst(n): Element n in the constant list
      TValue *element_n_of_constant_list = KBx(i);
      print_type(element_n_of_constant_list);
      // TValue x = *base;
      // print_type(base);
      printf("%p\n%p\n", register_A, element_n_of_constant_list);
      printf(" => OP_LOADK  /*	A Bx	R(A) := Kst(Bx)					*/");
    )
    vmcase(OP_GETTABUP,
      // Register A (specified in instruction field A)
      TValue *register_A = RA(i);
      print_type(register_A);
      // Register B or a constant index
      // TValue *register_B = RB(i);
      // print_type(register_B);
      // Register C or a constant index
      TValue *register_C_or_constant_index = RKC(i);
      print_type(register_C_or_constant_index);
      printf(" => OP_GETTABUP /*	A B C	R(A) := UpValue[B][RK(C)]			*/");
    )
    vmcase(OP_SETTABUP,
      // Register A (specified in instruction field A)
      TValue *register_A = RA(i);
      print_type(register_A);
      // Register B or a constant index
      TValue *register_B_or_constant_index = RKB(i);
      print_type(register_B_or_constant_index);
      // Register C or a constant index
      TValue *register_C_or_constant_index = RKC(i);
      print_type(register_C_or_constant_index);

      // TValue x = *base;
      // print_type(base);
      printf("%p\n%p\n%p\n", register_A, register_B_or_constant_index, register_C_or_constant_index);
      printf(" => OP_SETTABUP /*	A B C	UpValue[A][RK(B)] := RK(C)*/");
    )
    vmcase(OP_CALL,
      // printf(" => OP_CALL /*	A B C	R(A), ... ,R(A+C-2) := R(A)(R(A+1), ... ,R(A+B-1)) */");
    )
    vmcase(OP_RETURN,
      // printf(" => OP_RETURN /*	A B	return R(A), ... ,R(A+B-2)	(see note)	*/");
    )
    vmcase(OP_CLOSURE,
      // printf(" => OP_CLOSURE /*	A Bx	R(A) := closure(KPROTO[Bx])*/");
    )
  }
  printf("\n");
}

void luaV_execute (lua_State *L) {
  CallInfo *ci = L->ci;
  LClosure *cl;
  TValue *k;
  StkId base;
 newframe:  /* reentry point when frame changes (call/return) */
  lua_assert(ci == L->ci);
  cl = clLvalue(ci->func);
  k = cl->p->k;
  base = ci->u.l.base;
  /* main loop of interpreter */
  for (;;) {
    Instruction i = *(ci->u.l.savedpc++);
    StkId ra;
    int code = GET_OPCODE(i);
    print_opcode_detail(code, base, k, ci, i);
    ...
  }
}
...

```

#### Lua instructions detail

```c
/*===========================================================================
  We assume that instructions are unsigned numbers.
  All instructions have an opcode in the first 6 bits.
  Instructions can have the following fields:
	`A' : 8 bits
	`B' : 9 bits
	`C' : 9 bits
	'Ax' : 26 bits ('A', 'B', and 'C' together)
	`Bx' : 18 bits (`B' and `C' together)
	`sBx' : signed Bx

  A signed argument is represented in excess K; that is, the number
  value is the unsigned value minus K. K is exactly the maximum value
  for that argument (so that -max is represented by 0, and +max is
  represented by 2*max), which is half the maximum for the corresponding
  unsigned argument.
===========================================================================*/
```

#### lua_State

* [Lua 源码分析之线程对象 lua_State](https://blog.csdn.net/chenjiayi_yun/article/details/24304607)

```c
/*
** `per thread' state
every thread have many stack,
every stack is top field with type StkId (alias: TValue*)
every stack is stack_last field with type StkId (alias: TValue*)
every stack is stack field with type StkId (alias: TValue*)

stack 指向了一个可以动态增长的 TValue 数组, 数组的形式实现 stack
各种参数、局部变量、临时变量就临时的存活在这个动态 TValue 数组表示的栈上
*/
struct lua_State {
  CommonHeader;
  lu_byte status;
  StkId top;  /* first free slot in the stack */
  global_State *l_G;
  CallInfo *ci;  /* call info for current function */
  const Instruction *oldpc;  /* last pc traced */
  StkId stack_last;  /* last free slot in the stack */
  StkId stack;  /* stack base */
  int stacksize;
  unsigned short nny;  /* number of non-yieldable calls in stack */
  unsigned short nCcalls;  /* number of nested C calls */
  lu_byte hookmask;
  lu_byte allowhook;
  int basehookcount;
  int hookcount;
  lua_Hook hook;
  GCObject *openupval;  /* list of open upvalues in this stack */
  GCObject *gclist;
  struct lua_longjmp *errorJmp;  /* current error recover point */
  ptrdiff_t errfunc;  /* current error handling function (stack index) */
  CallInfo base_ci;  /* CallInfo for first level (C calling Lua) */
};
```

#### Lua 的 function、closure 和 upvalue

* [Lua 的 function、closure 和 upvalue](https://blog.csdn.net/soloist/article/details/319214)