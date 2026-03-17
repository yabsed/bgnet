# llvmmem-oopsla18.pdf 25-28페이지 한국어 번역

## 부록 A. LLVM과 GCC 모두에서 발생하는 종단 간 오컴파일
다음 C 프로그램은 GCC와 LLVM 모두에서 오컴파일된다.7

```c
$ cat b.c
void f(int *x, int *y) {}

$ cat a.c
#include <stdint.h>
#include <stdio.h>

void f(int *, int *);

int main(void) {
  int a = 0, y[1], x = 0;
  uintptr_t pi = (uintptr_t)&x;
  uintptr_t yi = (uintptr_t)(y + 1);
  int n = pi != yi;

    if (n) {
      a = 100;
      pi = yi;
    }

    if (n) {
      a = 100;
      pi = (uintptr_t)y;
    }

    *(int *)pi = 15;

    printf("a=%d x=%d\n", a, x);

    f(&x, y);

    return 0;
}
```

```text
$ clang-5.0 -Wall -O2 a.c b.c ; ./a.out
a=0 x=0
$ gcc-7 -Wall -O2 a.c b.c ; ./a.out
a=0 x=0
```

두 컴파일러가 만들어 내는 결과는 잘못되었다. 가능한 결과는 오직 두 가지뿐이며, 이는 `x`와 `y`가 연속해서 할당되는지(`n`이 false) 아니면 그렇지 않은지(`n`이 true)에 따라 달라진다.

- 경우 1: `n`이 true이면 프로그램은 반드시 다음을 출력해야 한다.

```text
a=100 x=0
```

- 경우 2: `n`이 false이면 프로그램은 반드시 다음을 출력해야 한다.

```text
a=0 x=15
```

주 7. 이 프로그램은 오컴파일이 일부 스택 레이아웃에서만 관찰되기 때문에, 플랫폼에 따라 서로 다른 결과를 낼 수 있다. 변수 `x`와 `y`의 선언 순서를 바꾸는 것만으로도 오컴파일을 관찰하기에 충분한 경우가 많다.

이 외의 어떤 출력도 허용되지 않는다. 두 컴파일러 모두에서 근본 원인은 정수에서 유도된 포인터를 충분히 보수적으로 다루지 않는 데 있다. 바람직한 포인터 최적화를 포기하지 않고서는 이 문제를 해결하기 어렵다. 이 프로그램은 두 컴파일러의 버그 추적 시스템에 모두 보고되어 있다. 이 논문에서 제시한 의미론은 이 문제를 해결하며, 우리의 프로토타입이 실제로 이 버그를 고친다는 것도 확인했다.

## 부록 B. LLVM이 오컴파일하는 안전한 Rust 프로그램
이 함수는 저수준 언어 기능을 사용하지만, 여전히 Rust의 안전한 부분집합에 속한다. 이 함수는 LLVM의 GVN 최적화에 있는 버그 때문에 오컴파일된다.

```rust
pub fn test(gp1: &mut usize, gp2: &mut usize, b1: bool, b2: bool) -> (i32, i32) {
    let mut g = 0;
    let mut c = 0;
    let y = 0;
    let mut x = 7777;
    let mut p = &mut g as *const _;

     {
          let mut q = &mut g;
          let mut r = &mut 8888;

          if b1 {
              p = (&y as *const _).wrapping_offset(1);
          }
          if b2 {
              q = &mut x;
          }

          *gp1 = p as usize + 1234;
          if q as *const _ == p {
              c = 1;
              *gp2 = (q as *const _) as usize + 1234;
              r = q;
          }
          *r = 42;
     }
     return (c, x);
}
```

이 함수는 먼저 `g`에 대한 참조를 `q`에 대입한다. 그 다음 값 `8888`을 담는 임시 객체를 만들고, `r`에는 그 객체에 대한 참조를 대입한다. 함수 `wrapping_offset`은 기저 객체의 경계를 벗어나도 안전하게 수행될 수 있는 포인터 산술을 수행한다(`inbounds`가 없는 LLVM의 `gep`와 동등하다). `b2`가 true이면 `q`에는 `x`에 대한 참조가 대입된다. 따라서 프로그램이 바로 뒤의 분기에도 들어가면 `r`에도 `x`에 대한 참조가 대입되고, 그 결과 이어지는 `r`을 통한 저장(`*r = 42;`)이 `x`의 값을 덮어쓴다.

`b1`과 `b2`를 모두 true로 두고 호출하면, 최적화된 이 코드는 `c = 1, x = 7777`을 반환한다. 이 결과는 불가능하다. 허용되는 결과는 `c = 0, x = 7777`과 `c = 1, x = 42`뿐이다.

이 오컴파일은 LLVM의 GVN 패스가 마지막 `if` 문의 조건을 이용해, 그 분기 안에서 `q`의 모든 사용을 잘못 `p`로 치환하면서 발생한다. 치환이 일어난 뒤에는 `r`이 이제 초기 임시 객체에 대한 참조이거나, `p`에 대입된 참조들 중 하나(`g`와 `y`만을 기반으로 한 참조)라고 가정되므로 `*r`이 `x`를 건드리지 않는다고 여겨진다. 따라서 `x = 7777`이 반환문까지 상수 전파되고, 이것이 잘못된 출력을 일으킨다.

## 부록 C. Coq 형식화와 증명
우리는 우리의 메모리 모델을 Coq로 형식화하고, 이 논문의 몇 가지 핵심 주장들을 증명했다. 코드는 `https://github.com/snu-sf/llvmtwin-coq`에서 구할 수 있다. 형식화를 단순하게 하기 위해 함수 호출과 반환은 생략했다.

### C.1 정의
메모리 모델은 `Memory.v` 파일에 명세되어 있다. 이 논문에서 제시한 설명과는 두 가지 점에서 다르다. 첫째, 간결함을 위해 address space를 지원하지 않는다. 둘째, 메모리는 마지막으로 사용된 블록 id를 유지하며, 이것은 할당 시 새로운 id를 만들 때 사용된다. twin block의 개수 `(|P|)`는 3으로 설정되어 있다.

메모리 블록의 well-formedness는 `Ir.MemBlock.wf`에 정의되어 있으며, 예를 들어 `|P|`가 항상 3이고 블록의 크기가 0보다 크다는 사실을 말한다. 메모리의 well-formedness는 `Ir.Memory.wf`에 정의되어 있으며, 예를 들어 존재하는 모든 메모리 블록이 well-formed이고, 살아 있는 블록들은 주소가 서로 겹치지 않는다는 사실을 말한다. 상태의 well-formedness는 `Ir.Config.wf`에 정의되어 있으며, 프로그램 카운터가 유효하고 메모리가 well-formed라는 식의 단언들을 포함한다. 우리는 어떤 명령을 실행하더라도 입력 상태의 well-formedness가 보존된다는 것을 증명했다.

small-step 의미론은 `SmallStep.v` 파일에 정의되어 있다. 입력 상태가 주어지면, 한 단계 실행의 결과는 다음 네 가지 중 하나가 될 수 있다. (1) 뒤따르는 명령의 실행이 성공하여 새로운 상태를 산출한 경우의 success, (2) 프로그램이 정의되지 않은 동작을 일으킨 경우의 UB, (3) 프로그램이 메모리 부족을 일으킨 경우의 OOM, (4) 프로그램이 종료된 경우의 halt이다.

메모리 블록 `l`에 관해 두 상태 `s`와 `s'`가 twin이라는 것은, `l`의 구성만 제외하면 두 상태가 같고, 그 `l`에 대해서는 `s'` 안의 주소들 `P'`가 `s` 안의 주소들 `P`의 순열이며, 활성화된 주소가 서로 다르다는 뜻이다.

어떤 블록의 주소는, 그 블록을 가리키는 포인터가 다음 명령들 가운데 하나의 인수로 주어질 때 관찰된 것으로 간주된다. 즉 `ptrtoint`, `psub`, `icmp`가 그것이다. 또한 뒤의 두 명령의 경우에는 다른 인수가 물리 포인터여야 한다. 관찰되지 않은 블록을 가리키는 포인터는 guessed pointer라고 한다.

### C.2 증명
우리는 Coq에서 다음 정리들을 증명했다.

- 정리 C.1 (트윈 할당은 포인터 추측을 금지한다). `l`에 관해 twin인 두 상태 `s`와 `s'`가 주어졌고, 다음 명령이 `l`에 대한 guessed pointer를 역참조한다면, `s`와 `s'` 중 적어도 하나의 실행은 UB를 일으킨다.
- 정리 C.2 (Malloc). `malloc`은 `NULL` 포인터를 반환하거나, 아니면 논리 포인터 `Log(l, o)`를 반환한다. 또한 small-step 실행이 산출하는 상태들은 모두 `l`에 관해 twin이다.
- 정리 C.3. `l`에 관해 twin인 두 상태 `s`와 `s'`가 주어졌다고 하자. 다음 명령이 guessed pointer를 역참조하지도 않고 블록 `l`을 관찰하지도 않는다면, `s`와 `s'`의 small-step 실행은 (1) 종료하거나 UB 또는 OOM을 일으키거나, (2) 성공하며 `s`의 후속 상태들과 `s'`의 후속 상태들이 서로 twin이 된다.

이제 특정 명령들이 할당/해제 함수의 앞뒤로 자유롭게 이동될 수 있음을 보인다.

- 정리 C.4 (명령 재배치). `icmp eq`, `icmp ule`, `psub`, `inttoptr`, `ptrtoint`, `gep` 명령은 `malloc`과 `free`를 가로질러 양방향(위쪽과 아래쪽)으로 이동될 수 있다. 또한 이런 재배치를 수행해 `P`로부터 얻은 프로그램 `P'`는 `P`와 동등하다.

마지막으로, 우리는 GVN의 건전성을 위한 충분조건을 증명한다.

- 정리 C.5 (포인터에 대한 GVN의 건전성). 5절에서 제시한 네 가지 조건, 즉 포인터 `p`를 다른 포인터 `q`로 치환해도 건전한 경우를 규정하는 조건들은 올바르다. 다시 말해, 그 포인터 치환은 refinement이다.

## 참고문헌
- Frédéric Besson, Sandrine Blazy, and Pierre Wilke. 2014. A Precise and Abstract Memory Model for C Using Symbolic Values. In APLAS. https://doi.org/10.1007/978-3-319-12736-1_24
- Frédéric Besson, Sandrine Blazy, and Pierre Wilke. 2015. A Concrete Memory Model for CompCert. In ITP. https://doi.org/10.1007/978-3-319-22102-1_5
- Frédéric Besson, Sandrine Blazy, and Pierre Wilke. 2017a. CompCertS: A Memory-Aware Verified C Compiler Using Pointer as Integer Semantics. In ITP. https://doi.org/10.1007/978-3-319-66107-0_6
- Frédéric Besson, Sandrine Blazy, and Pierre Wilke. 2017b. A Verified CompCert Front-End for a Memory Model Supporting Pointer Arithmetic and Uninitialised Data. Journal of Automated Reasoning (03 Nov 2017). https://doi.org/10.1007/s10817-017-9439-z
- David Chisnall, Justus Matthiesen, Kayvan Memarian, Peter Sewell, and Robert N. M. Watson. 2016. C memory object and value semantics: the space of de facto and ISO standards. https://www.cl.cam.ac.uk/~pes20/cerberus/notes30.pdf
- Arie Gurfinkel and Jorge A. Navas. 2017. A Context-Sensitive Memory Model for Verification of C/C++ Programs. In SAS. https://doi.org/10.1007/978-3-319-66706-5_8
- Chris Hathhorn, Chucky Ellison, and Grigore Roşu. 2015. Defining the Undefinedness of C. In PLDI. https://doi.org/10.1145/2737924.2737979
- Jeehoon Kang, Chung-Kil Hur, William Mansky, Dmitri Garbuzov, Steve Zdancewic, and Viktor Vafeiadis. 2015. A Formal C Memory Model Supporting Integer-pointer Casts. In PLDI. https://doi.org/10.1145/2737924.2738005
- Robbert Krebbers. 2013. Aliasing Restrictions of C11 Formalized in Coq. In CPP. https://doi.org/10.1007/978-3-319-03545-1_4
- Robbert Krebbers and Freek Wiedijk. 2015. A Typed C11 Semantics for Interactive Theorem Proving. In CPP. 15-27. https://doi.org/10.1145/2676724.2693571
- Juneyoung Lee, Yoonseung Kim, Youngju Song, Chung-Kil Hur, Sanjoy Das, David Majnemer, John Regehr, and Nuno P. Lopes. 2017. Taming Undefined Behavior in LLVM. In PLDI. https://doi.org/10.1145/3062341.3062343
- Xavier Leroy. 2009. Formal Verification of a Realistic Compiler. Commun. ACM 52, 7 (July 2009), 107-115. https://doi.org/10.1145/1538788.1538814
- Xavier Leroy and Sandrine Blazy. 2008. Formal Verification of a C-like Memory Model and Its Uses for Verifying Program Transformations. Journal of Automated Reasoning 41, 1 (Jul 2008), 1-31. https://doi.org/10.1007/s10817-008-9099-0
- Nuno P. Lopes, David Menendez, Santosh Nagarakatte, and John Regehr. 2015. Provably Correct Peephole Optimizations with Alive. In PLDI. https://doi.org/10.1145/2737924.2737965
- Kayvan Memarian, Justus Matthiesen, James Lingard, Kyndylan Nienhuis, David Chisnall, Robert N.M. Watson, and Peter Sewell. 2016. Into the depths of C: elaborating the de facto standards. In PLDI. https://doi.org/10.1145/2908080.2908081
- Kayvan Memarian and Peter Sewell. 2016a. Clarifying the C memory object model (revised version of WG14 N2012). https://www.cl.cam.ac.uk/~pes20/cerberus/notes64-wg14.html
- Kayvan Memarian and Peter Sewell. 2016b. N2090: Clarifying Pointer Provenance (Draft Defect Report or Proposal for C2x). https://www.cl.cam.ac.uk/~pes20/cerberus/n2090.html
- The LLVM Project. 2018. LLVM Language Reference Manual. https://llvm.org/docs/LangRef.html
- Raimondas Sasnauskas, Yang Chen, Peter Collingbourne, Jeroen Ketema, Jubi Taneja, and John Regehr. 2017. Souper: A Synthesizing Superoptimizer. CoRR abs/1711.04422 (2017). http://arxiv.org/abs/1711.04422
- Jaroslav Ševčík, Viktor Vafeiadis, Francesco Zappa Nardelli, Suresh Jagannathan, and Peter Sewell. 2013. CompCertTSO: A Verified Compiler for Relaxed-Memory Concurrency. J. ACM 60, 3, Article 22 (June 2013), 22:1-22:50 pages. https://doi.org/10.1145/2487241.2487248
- Wei Wang, Clark Barrett, and Thomas Wies. 2017. Partitioned Memory Models for Program Analysis. In VMCAI. https://doi.org/10.1007/978-3-319-52234-0_29
- Jianzhou Zhao, Santosh Nagarakatte, Milo M.K. Martin, and Steve Zdancewic. 2012. Formalizing the LLVM Intermediate Representation for Verified Program Transformations. In POPL. https://doi.org/10.1145/2103656.2103709
