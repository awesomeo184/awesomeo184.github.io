---  
emoji: 📝  
title: '서버 메모리 부족 문제 해결기'   
date: '2022-11-20 23:00:00'  
author: 어썸오  
tags: JVM OOME
categories: projects
--- 

우테코 프로젝트 제약상 S3같은 스토리지 서비스를 사용할 수 없어 직접 EC2에 이미지 스토리지 서버를 구축해 서비스를 운영하고 있었습니다.

아래 인프라 구조에서 볼 수 있듯이 클라이언트에서 이미지 서버로 이미지 조회를 요청하도록 설계하였습니다.

![server_arch](server_arch.png)

<br>

그런데 이미지 조회에 대한 부하테스트 중, 테스트를 시작하자마자 엄청난 에러가 쏟아지는 것을 확인했습니다. 사용자가 적다보니 운영에서는 미처 발견하지 못한 문제였습니다.

// 최대 힙크기 64MB일 때 인스펙터 사진
![full_gc](full_gc.png)

![heap_space_error](heap_space_error.png)

위 사진에서 확인할 수 있듯이 테스트가 시작되자마자 Full GC가 비정상적으로 발생하고, 힙 공간 부족으로 인한 OutOfMemoryError가 발생하고 있었습니다.

이미지 서버는 t4.micro를 사용하고 있었고 메모리 크기는 1GB 였습니다. 최대 힙 크기 설정을 따로 해주지 않았기 때문에 기본 값을 사용하고 있었는데 확인해보니 64MB였습니다.


단순히 최대 힙 크기를 늘리면 해결되겠지 생각하고 최대 힙 크기를 256MB로 올렸습니다.
256MB인 이유 - 사용자 이용 패턴, 레이지 로드라 한 번 접속할 떄 이미지 3개를 넘지 않음

애플리케이션 띄우고 top으로 현재 메모리 상태 살펴보니 600정도 남음. 절반 정도 할당하면 될거라고 생각
// top 이미지

혹시 몰라 스왑 메모리 늘림

256으로 늘리고 테스트 돌렸더니 아직도 OOME

![full_gc](full_gc.png)

![heap_space_error](heap_space_error.png)


이미지는 거의 변하지 않고, 조회가 많음. 캐시로 이 부분을 해결하고 있음. 즉 WAS로 이미지 조회 요청이 몰리는 경우는 잘 발생하지 않음.

-> 예외적인 경우를 대비하여, 응답 속도는 조금 느리더라도 서버가 죽는 것만은 막아보자

512MB로 올림. 프로세스가 죽어버림

// 저널컨트롤 명령어
// 시스템 로그 사진

OOM Killer가 JVM 프로세스를 죽여버린 것을 확인

[OOM Killer는 어떤 방식으로 죽일 프로세스를 찾는지에 대해]

[리눅스 시스템 로그]

응답 시간이 지연되더라도 서버가 죽는 것 막자. 스레드 수 제한.


[메모리 분석하는법]

-XX:NativeMemoryTracking=summary

```

Native Memory Tracking:

Total: reserved=2082907KB +15KB, committed=448079KB +79KB

-                 Java Heap (reserved=524288KB, committed=262144KB)
                            (mmap: reserved=524288KB, committed=262144KB)

-                     Class (reserved=1118043KB, committed=77903KB)
                            (classes #14122)
                            (  instance classes #13293, array classes #829)
                            (malloc=1883KB #32845 +19)
                            (mmap: reserved=1116160KB, committed=76020KB)
                            (  Metadata:   )
                            (    reserved=67584KB, committed=66420KB)
                            (    used=65294KB +23KB)
                            (    free=1126KB -23KB)
                            (    waste=0KB =0.00%)
                            (  Class space:)
                            (    reserved=1048576KB, committed=9600KB)
                            (    used=8935KB)
                            (    free=665KB)
                            (    waste=0KB =0.00%)

-                    Thread (reserved=102637KB, committed=5125KB)
                            (thread #50)
                            (stack: reserved=102400KB, committed=4888KB)
                            (malloc=180KB #302)
                            (arena=58KB #99)

-                      Code (reserved=249109KB +9KB, committed=23805KB +73KB)
                            (malloc=1421KB +9KB #7255 +31)
                            (mmap: reserved=247688KB, committed=22384KB +64KB)

-                        GC (reserved=57974KB +1KB, committed=48246KB +1KB)
                            (malloc=5710KB +1KB #8969 +26)
                            (mmap: reserved=52264KB, committed=42536KB)

-                  Compiler (reserved=178KB, committed=178KB)
                            (malloc=45KB #517)
                            (arena=133KB #5)

-                  Internal (reserved=1039KB, committed=1039KB)
                            (malloc=1007KB #1909)
                            (mmap: reserved=32KB, committed=32KB)

-                     Other (reserved=8275KB, committed=8275KB)
                            (malloc=8275KB #17)

-                    Symbol (reserved=17138KB, committed=17138KB)
                            (malloc=14700KB #176682 +3)
                            (arena=2437KB #1)

-    Native Memory Tracking (reserved=3663KB +5KB, committed=3663KB +5KB)
                            (malloc=18KB +3KB #235 +50)
                            (tracking overhead=3645KB +2KB)

-               Arena Chunk (reserved=178KB, committed=178KB)
                            (malloc=178KB)

-                   Logging (reserved=4KB, committed=4KB)
                            (malloc=4KB #188)

-                 Arguments (reserved=19KB, committed=19KB)
                            (malloc=19KB #497)

-                    Module (reserved=194KB, committed=194KB)
                            (malloc=194KB #2312)

-              Synchronizer (reserved=162KB, committed=162KB)
                            (malloc=162KB #1363)

-                 Safepoint (reserved=8KB, committed=8KB)
                            (mmap: reserved=8KB, committed=8KB)

```





[조금 더 튜닝]

jcmd 명령으로 가상머신 메모리 사용량 확인 가능

> jcmd {PID} VM.native_memory baseline
> jcmd {PID} VM.native_memory summary.diff



sudo jstat -gccapacity 17692
```
NGCMN    NGCMX     NGC     S0C   S1C       EC      OGCMN      OGCMX       OGC         OC       MCMN     MCMX      MC     CCSMN    CCSMX     CCSC    YGC    FGC   CGC
     0.0 262144.0 157696.0    0.0 1024.0 156672.0        0.0   262144.0   104448.0   104448.0      0.0 1124352.0  86128.0      0.0 1048576.0  10536.0    958   140   519
```


```
ubuntu@ip-192-168-1-212:~$ sudo jstat -gc 17692 1000
 S0C    S1C    S0U    S1U      EC       EU        OC         OU       MC     MU    CCSC   CCSU   YGC     YGCT    FGC    FGCT    CGC    CGCT     GCT
 0.0   1024.0  0.0   1024.0 156672.0 26624.0   104448.0   89281.0   86128.0 83693.6 10536.0 9725.2    958    5.534  140    18.955  519     7.467   31.956
 0.0   1024.0  0.0   1024.0 156672.0 26624.0   104448.0   89281.0   86128.0 83693.6 10536.0 9725.2    958    5.534  140    18.955  519     7.467   31.956
 0.0   1024.0  0.0   1024.0 156672.0 26624.0   104448.0   89281.0   86128.0 83693.6 10536.0 9725.2    958    5.534  140    18.955  519     7.467   31.956
 0.0   1024.0  0.0   1024.0 156672.0 26624.0   104448.0   89281.0   86128.0 83693.6 10536.0 9725.2    958    5.534  140    18.955  519     7.467   31.956
 0.0   1024.0  0.0   1024.0 156672.0 26624.0   104448.0   89281.0   86128.0 83693.6 10536.0 9725.2    958    5.534  140    18.955  519     7.467   31.956
 0.0   1024.0  0.0   1024.0 156672.0 26624.0   104448.0   89281.0   86128.0 83693.6 10536.0 9725.2    958    5.534  140    18.955  519     7.467   31.956
 0.0   1024.0  0.0   1024.0 156672.0 26624.0   104448.0   89281.0   86128.0 83693.6 10536.0 9725.2    958    5.534  140    18.955  519     7.467   31.956
 0.0   1024.0  0.0   1024.0 156672.0 26624.0   104448.0   89281.0   86128.0 83693.6 10536.0 9725.2    958    5.534  140    18.955  519     7.467   31.956
 0.0   1024.0  0.0   1024.0 156672.0 26624.0   104448.0   89281.0   86128.0 83693.6 10536.0 9725.2    958    5.534  140    18.955  519     7.467   31.956
 0.0   1024.0  0.0   1024.0 156672.0 26624.0   104448.0   89281.0   86128.0 83693.6 10536.0 9725.2    958    5.534  140    18.955  519     7.467   31.956
 0.0   1024.0  0.0   1024.0 156672.0 26624.0   104448.0   89281.0   86128.0 83693.6 10536.0 9725.2    958    5.534  140    18.955  519     7.467   31.956
 0.0   1024.0  0.0   1024.0 156672.0 26624.0   104448.0   89281.0   86128.0 83693.6 10536.0 9725.2    958    5.534  140    18.955  519     7.467   31.956
```



young 영역과 old 영역. 객체의 특성
NewRatio=4

```
ubuntu@ip-192-168-1-212:~$ sudo jstat -gc 17978 1000
 S0C    S1C    S0U    S1U      EC       EU        OC         OU       MC     MU    CCSC   CCSU   YGC     YGCT    FGC    FGCT    CGC    CGCT     GCT
 0.0   7168.0  0.0   7168.0 48128.0  20480.0   206848.0   51078.3   86004.0 83678.3 10572.0 9718.6    877    4.902  220    21.972  595     8.433   35.308
 0.0   7168.0  0.0   7168.0 48128.0  20480.0   206848.0   51078.3   86004.0 83678.3 10572.0 9718.6    877    4.902  220    21.972  595     8.433   35.308
 0.0   7168.0  0.0   7168.0 48128.0  20480.0   206848.0   51078.3   86004.0 83678.3 10572.0 9718.6    877    4.902  220    21.972  595     8.433   35.308
 0.0   7168.0  0.0   7168.0 48128.0  20480.0   206848.0   51078.3   86004.0 83678.3 10572.0 9718.6    877    4.902  220    21.972  595     8.433   35.308
 0.0   7168.0  0.0   7168.0 48128.0  20480.0   206848.0   51078.3   86004.0 83678.3 10572.0 9718.6    877    4.902  220    21.972  595     8.433   35.308
 0.0   7168.0  0.0   7168.0 48128.0  20480.0   206848.0   51078.3   86004.0 83678.3 10572.0 9718.6    877    4.902  220    21.972  595     8.433   35.308

```






