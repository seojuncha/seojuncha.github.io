+++
date = '2025-02-06'
title = "테스트 벤치와 Verilog 개발 환경 설정: Icarus Verilog와 cocotb"
tags = ["verilog", "testbench", "cocotb", "icarus verilog", "gtkwave"]
+++

다른 소프트웨어 프로그래밍 언어와 마찬가지로 Verilog로 작성한 코드 역시 의도한대로 동작하는지 검증하는 과정이 필요합니다.
Verilog는 크게 두가지 방식으로 검증할 수 있습니다.
- 하드웨어 기반 검증
- 소프트웨어 기반 검증

하드웨어 기반 검증 방식은 Verilog코드를 실제 하드웨어에 업로드하여 입력신호에 따른 출력신호를 확인하는 방법입니다.
이에 반해, 소프트웨어 기반 검증 방식은 별도의 하드웨어 없이 순수 소프트웨어만으로 코드의 동작을 검증하는 방법입니다.
하드웨어 기반의 검증방식은 하드웨어가 필요할 뿐아니라 신호 변화를 디버깅하기 어렵기 때문에 하드웨어 업로드 전에 소프트웨어 기반 검증 방식을 사용합니다.
이런 일련의 소프트웨어 기반 검증 과정을 수행하기 위한 코드를 ***테스트벤치\(Testbench\)***라고 하고 테스트 벤치를 실행하는 환경을 ***시뮬레이터\(Simulator\)***라고 합니다.

이번 포스팅에서 **Verilog 테스트벤치 개념**과 Ubuntu와 MacOS에서 **시뮬레이션 환경 설정**에 관해 알아보겠습니다.
각자의 개발환경에 맞는 시뮬레이터를 설정하고 테스트벤치를 작성해야 하는 이유를 이해하여야만 앞으로 진행할 Verilog 프로그래밍 관련 포스팅을 원활히 따라갈 수 있습니다.

# 테스트벤치의 개념과 필요성, 그리고 소프트웨어 테스트 방식과의 비교
테스트벤치는 Verilog로 작성된 회로를 실제 하드웨어로 구현하지 않고도 검증할 수 있도록 지원하는 환경입니다. 이를 통해 설계한 회로의 동작을 시뮬레이션하고, 입력을 제공하며, 출력을 확인할 수 있습니다.

예상할 수 있듯이 테스트 벤치의 작성은 **소프트웨어 기반 검증** 방식입니다. 

소프트웨어 개발에서는 출력 값을 확인하거나 디버깅 도구를 사용하여 프로그램의 실행 흐름을 분석할 수 있습니다. 하지만 하드웨어 설계에서는 일반적인 프로그래밍 방식으로 값을 출력하거나 직접 중단점(Breakpoint)을 설정할 수 없습니다.

대신, 테스트벤치를 사용하면 시뮬레이션 환경에서 하드웨어 회로의 동작을 검증할 수 있습니다. 즉, 소프트웨어 프로그래머가 단위 테스트(Unit Test)나 디버깅을 수행하는 것과 유사한 방식으로 Verilog 코드를 검증할 수 있는 방법입니다.

소프트웨어 디버깅/테스트 방식과 비교하면 아래와 같은 주요 차이점이 있습니다.
|비교 항목|C|Python|Verilog 테스트벤치|
|:-:|:-:|:-:|:-:|
|코드 실행 환경|CPU|인터프리터|시뮬레이터|
|디버깅 방식|printf, gdb|print, pdb|$display, waveform|
|출력 확인 방법|터미널 출력|터미널 출력|터미널 출력, waveform파일|

# Verilog 시뮬레이터 소개
Verilog 테스트벤치는 기본적으로 하드웨어 동작을 시뮬레이션 하기 위한 코드이고, 이 코드를 실행할 환경이 필요합니다.
이 환경이 바로 시뮬레이터 입니다. 시뮬레이터에 따라 테스트벤치를 작성하는 언어는 다를 수 있지만, 결과적으로 **특정 시점의 입출력신호를 파악할 수 있도록 하는 것이 주요 목적**입니다.

> 참고로, 테스트벤치는 하드웨어에 반영되지 않는 오직 테스트를 위한 코드입니다.

주로 널리 사용되는 Verilog 시뮬레이터는,
- `Icarus Verilog`
- `Verilator`
- `ModelSim`
- 그 외 FPGA 벤더(예: Xillinx) 제공 시뮬레이터

가 있습니다.

향후 당분간의 포스팅에서는 `Icarus Verilog`를 사용할 것이므로 Icarus Verilog를 설치하는 방법을 소개하겠습니다.

> ***꼭 시뮬레이터를 사용해서 테스트벤치를 실행해야 하는가?***  
> 
> 그렇지는 않습니다. 하지만 추천하지 않는 방법입니다.  
> 다른 포스팅에서 다루겠지만, 시뮬레이터를 사용하지 않고 Verilog 코드만으로 테스트벤치를 작성하여 테스트 할 수도 있습니다. 하지만, 설계가 복잡해지면 한계가 명확해집니다.
> 
> **하드웨어 검증의 핵심은 신호와 타이밍 분석**이기 때문입니다.
>
> 소프트웨어의 경우 순차적 실행이므로 변수의 값이나 함수의 흐름을 출력해서 변화를 살펴볼 수 있지만, 하드웨어는 병렬 실행이 가능하기 때문에 단순히 프로그램의 실행 순서에 따른 변화가 아니라 특정 시점에서 변화하는 신호를 분석해야 합니다.  
> 이런 특성으로 인해 기존의 소프트웨어 디버깅 방법을 사용하는 것 보단 waveform viewer를 사용한 파형분석이 요구됩니다.

## 시뮬레이션 설치 환경
아래 설명할 설치 가이드는 아래의 환경에서 검증되었습니다.
### Ubuntu
- OS 버전
  - 22.04
### MacOS
- 프로세서
  - Apple Sillicon M2
- OS 버전
  - Sequoia 15.1.1

## Icarus Verilog란?
[Icarus Verilog](https://en.wikipedia.org/wiki/Icarus_Verilog)는 다양한 운영체제를 지원하는 [오픈소스](https://github.com/steveicarus/iverilog) Verilog 시뮬레이터 입니다. 아마 가장 유명하고 널리 사용되는 Verilog 시뮬레이터 중 하나일 것 같습니다. 

[Icarus Verilog 공식 문서](https://steveicarus.github.io/iverilog/)를 통해 자세한 사용법을 익힐 수 있습니다.

### Ubuntu에 Icarus Verilog 설치
- 설치
```shell
$ sudo apt install iverilog
```
- 설치 확인
```shell
$ iverilog -v
```

### MacOS에 Icarus Verilog 설치

- 설치
```shell
$ brew install icarus-verilog
```
- 설치 확인
```shell
$ iverilog -v
```

## Verilog & VHDL 테스트 프레임워크: cocotb
[cocotb](https://www.cocotb.org/)는 다양한 Verilog 시뮬레이터를 [벡엔드](https://docs.cocotb.org/en/stable/simulator_support.html)로 사용하는 파이썬 기반 테스트 프레임워크 입니다.
파이썬 기반으로 작성되었기 때문에 테스트벤치를 Verilog로 작성할 필요없이 파이썬으로 쉽고 빠르게 작성하고 시뮬레이션 할 수 있습니다.

### pip를 사용한 cocotb 설치

cocotb는 파이썬 기반 프레임워크 이므로 `pip`를 사용해서 설치할 수 있습니다. 따라서, OS와 무관하게 아래의 방법을 따릅니다.

- 설치
```shell
$ pip3 install cocotb
```
- 설치 확인
```shell
$ pip3 list | grep cocotb
cocotb              1.9.2

$ python3 -c "import cocotb; print(cocotb.__version__)"
1.9.2

$ cocotb-config --help
```

# 파형 분석 툴: GTKWave
[GTKWave](https://gtkwave.sourceforge.net/) 는 하드웨어 신호 [파형(Waveform)](https://en.wikipedia.org/wiki/Waveform)을 직접 눈으로 볼 수 있는 소프트웨어 입니다. 
Verilog 시뮬레이터가 생성한 파형 파일(예: `FST`, `VCD`)을 읽어서 각 모듈의 신호 변화를 타임스탬프에 맞춰 그려줍니다.

Verilog 디버깅 시에 필수로 사용되므로 [사용 매뉴얼](https://gtkwave.sourceforge.net/gtkwave.pdf)을 참고하여 익숙해지는 것이 좋습니다.

## MacOS 에서 GTKWave 빌드 및 실행
MacOS에서 `brew`를 사용해서 GTKWave를 설치하는 경우, GTKWave가 정상 실행되지 않습니다. 따라서, 아래의 방법을 사용해 직접 빌드하여 사용하는 방식을 추천합니다.

```shell
$ brew install desktop-file-utils shared-mime-info \
               gobject-introspection gtk-mac-integration \
               meson ninja pkg-config gtk+3 gtk4
$ git clone https://github.com/gtkwave/gtkwave.git
$ cd gtkwave
$ meson setup build
$ meson compile -C build
$ cd build/src
$ ./gtkwave
```

### Ubuntu 에서 GTKWave 설치하기
Ubuntu에서는 패키지 관리자를 사용하여 간단하게 설치할 수 있습니다.
```shell
$ sudo apt install gtkwave
$ gtkwave --version
```

# 결론
본 포스팅에서는 테스트벤치를 사용한 Verilog로 작성한 코드의 검증, 디버깅에 관한 내용과 더불어 시뮬레이터를 간단히 소개했습니다. 소프트웨어 프로그램과 마찬가지로 **프로그램의 검증과 디버깅**은 매우 중요한 과정입니다.

Icarus Verilog를 중심으로 테스트 환경을 구성했지만, 필요에 따라 Verilator 또는 ModelSim을 사용할 수도 있습니다. 앞으로 진행될 Verilog 관련 포스팅에서는 시뮬레이터와 테스트 프레임워크의 활용법을 차근차근 소개하겠습니다.

# 연관 주제
- [소프트웨어 개발자가 Verilog를 배워야 하는 이유](posts/introduction-verilog-for-programmer)