# Chapter 3 : The Graphics Processing Unit

## 0. Intro

* 20년도 더 전에, NVIDIA 의 GeForce 256 이라는 그래픽 카드가 나오면서 GPU 시장은 격변기를 겪게 됨. 왜냐하면 GeForce 256 은 기존의 *하드웨어 래스터라이제이션* 만 가능했던 칩들과는 다르게 **정점 처리**를 하드웨어에서 지원해주는 기능이 있었기 때문.
  * NVIDIA 는 이 카드를 기점으로 **GPU (Graphics Processing Unit)** 이라는 용어를 사용하게 함.

* 이 이후로 복잡한 고정된 파이프라인 설계 지향의 GPU (Fixed Pipeline) 이 나오게 되고, 그리고 Shader 의 추가로 **프로그래밍 가능하고, 유연한** 파이프라인의 설계 지향의 GPU 들이 나오게 됨.
* GPU 는 **병렬화 할 수 있는 작은 태스크들**을 아주 빨리 처리할 수 있도록 특별히 설계되었음. 또한 *z-buffer* 와 같은 특수한 버퍼를 처리하기 위해, 텍스쳐 이미지 혹은 다른 버퍼에 신속하게 접근할 수 있도록 하드웨어가 설계되었음.
  * 어떻게 처리되는가는 `Chapter 23` 에서 설명함.
* 무수한 정점들이 한꺼번에 처리되기 위해서는 쉐이더 및 여러가지 연산을 무수히 호출해서 처리하게 하지 않으면 안됨. 이로 인해서 발생하는 지연을 **레이턴시 (Latency)** 라고 함. 일반적으로 레이턴시는 접근할 버퍼가 처리 유닛에서 멀게 되면 증가함.
  * 자세한 설명은 `Section 23.3` 에서 설명.
* 메모리 칩에 저장된 정보들은 각 유닛에 부착된 레지스터에 저장된 정보들보다 접근하는 데 더 많은 시간이 걸림.
  * 자세한 설명은 `Section 18.6` 에서 설명.

## 1. Data-Parallel Architectures

* CPU 는 매우 다양한 자료구조 및 매우 큰 코드들을 효율적으로 처리하기 위하도록 설계되었다. 그런데 CPU 는 코어가 그렇게 많지 않고, 대체로 순서대로 코드들을 처리하다 보니 속도가 느려질 수 있다. 이 속도 저하를 보다 줄이기 위해, 다양한 방법들이 사용되고 있다.

  * *SIMD (Single Instruction Multi Data)* 를 사용해서 속도를 잠재적으로 높일 수 있지만 한계가 있다.
  * 하드웨어에서 사용하는 방법으로는 *분기 예측*, *명령어 재배열*, *레지스터 변경*, *캐시 프리패칭* 등이 있다. `[715]`

* GPU 는 **개개의 코어가 CPU 보다는 많이 느리지만, 많은 코어를 배치 가능하다는 점에서 병렬화된 태스크를 전체적으로는 CPU 보다 더 빠르게** 처리할 수 있다. `[462]`

  * GPU 의 각각의 코어를 **Shader Core** 라고 한다.

* 렌더링 작업 시, 일반 메모리에 존재하는 텍스쳐 등을 가지고 쉐이더를 처리할 때 버퍼 접근 시의 Stalling 을 줄이기 위해, GPU 의 각 프로세서 단에서 텍스쳐의 Fragment 을 레지스터에 적재하여 처리하게 할 수도 있다. 이를 사용함으로써 Stalling 을 줄이고 처리속도를 늘일 수 있다.

* **GPU 는 SIMD 을 적극적으로 활용한다.** *SIMD* 는 데이터 군(群) 과 명령어를 나눠서 데이터 독립적으로 이론 상 몇배의 속도로 동시에 처리할 수 있다. 

  * 어느 한 *Fragment* 에 대한 각각의 픽셀 쉐이더 호출의 작업을 **Thread** 라고 부를 수 있다.
  * 동일한 쉐이더 프로그램을 사용하는 *Thread* 들은 일정 개수들로 묶어서 **Warp / Wavefront** 라고 부를 수 있다. 이 Warp 는 GPU 쉐이더 코어군에 의해 묶여서 실행될 수 있으며, 실행 순서를 배정받는다.
  * 따라서 Warp 의 Thread 들은 동시에 실행되기 위해 (Warp 단위로) **SIMD lane** 에 맵핑된다.

* GPU 는 CPU 와는 다르게 완전히 *SIMD* 을 사용하도록 하드웨어가 설계되었기 때문에, `if` 와 같은 브랜칭 명령어에는 대단히 취약하다. 왜냐면 Warp 의 스레드들 중에서, 하나는 `then` 구문을, 다른 하나는 `else` 구문을 택할 수 있기 때문이다.

  * 이를 **Thread Divergence** 라고 부르며, 이 분기문을 수행하기 위해 모든 Thread 들은 `then` 과 `else` 을 전부 다 수행해서 분기에 맞지 않는 결과를 버려야 한다.
    * 최근에는 이 `if` 구문을 컴파일 시 최적화하는 경우도 더러 존재한다.

## 2. GPU Pipeline Overview

* GPU 는 **Logical model** 과 **Physical Model** 로 나뉜다. 이는 프로그래머에게 보다 더 추상화된 모델을 제공함으로써 편하게 그래픽스 작업을 하기 위함이라고 한다. *Logical Model* 과 *Physical Model* 은 항상 같으리라는 법은 없으며, Physical Model 은 하드웨어 벤더에 의해 각기 다 다를 수도 있다.
  * *Logical Model* 은
    `VS` `Tessellation` `GS` `Clipping` `Screen Mapping` `Triangle Setup & Traversal` `Pixel Shader` `Merger` 
    의 스테이지 구성으로 나뉜다.
* *Merger Stage* 는 일반적으로 Fixed-Function 스테이지이다. 여기서는 **세팅에 의한 컬러 조정, z-buffer, blend, stencil test** 등을 담당한다.
  * Early depth-testing 과 같은, 래스터라이저 이전의 고정 함수 단계를 제외하면 거의 모든 고정 함수 상태의 프로세싱은 Merger 에서 수행해도 된다고 봐도 될 듯.

## 3. The Programmable Shader Stage

* *Draw Call* 은 그래픽스 API 로 하여금 프리미티브 (토폴로지) 를 그리게 하는 것을 말한다. 인풋 어셈블리와 쉐이더를 돌리는 형태로 드로우 콜이 실행된다.
* 각각의 쉐이더 스테이지이 입력은 두 가지 종류로 나뉜다. 하나는 **Uniform** 이고 또 하나는 **Varying** 이다.
  * **Uniform** 은 하나의 드로우 콜에서 고정된 값을 가지는 입력 값들이다. Uniform 은 Varying 과는 다르게 각 정점마다 인풋이 다 다를 필요가 없기 때문에 꽤 많은 값의 허용범위를 가진다.
  * **Varying** 은 인풋 어셈블리 혹은 래스터라이제이션 이후의 각각의 정점이 가진 값들의 입력 값을 말한다. *Uniform* 과는 달리 각 정점마다 각기 다른 값들이 들어가야 하기 때문에 허용범위는 꽤 작을 수 있다.
* 쉐이더를 실행하는 추상화된 그래픽스 VM (Virtual Machine) 은 각기 다른 입력과 출력 값들에 대해 특별한 레지스터를 사용하고 있다. *Uniform* 의 경우에는 **Constant Register** 라는 것을 사용하고 있는데, 이 *Constant Register* 는 입력될 수 있는 값들의 개수 범위가 크다. 그리고 **Temporary Resgiter** 는 쉐이더 VM 에서 다양하게 사용될 수 있는 레지스터이다.
  * `Varying` `Uniform` `Constant Regiter` `Temporary Register` `VM` `Texture` 의 구성에 관한 도면은 책의 `p36` 을 보라.
* **Intrinsic Functions** 은 Shader VM 에서 일반적인 연산 혹은 간단한 연산들을 수행할 때 GPU 에 의해 최적화 될 수 있는 함수들을 말한다.
* Shader 에서의 분기 예측 (`if` `for`) 은 정적인가 동적인가에 따라 두 종류로 나뉠 수 있다.
  * **Static Flow Control** 은 *Uniform* 값에 의해서 결정이 된다. 드로우 콜을 할 때, 유니폼의 값은 변하지 않기 때문에 (보통) 따라서 성능이 떨어지는 일이 없을 것이다.
  * **Dynamic Flow Control** 은 Varying 값에 의해서 결정이 되는데, 이는 코드가 다 다르게 결정이 될 수 있다는 것을 내포한다. 따라서 성능이 떨어질 가능성이 존재한다.

## 5. The Vertex Shader

* Vertex Shader 는 **첫번째로 프로그래밍이 가능한 쉐이더 스테이지**이다.
  * *VS* 에서는 단순히 어셈블러에서 취합해서 가져온 정점만을 독립적으로 다른 정점으로 변환하기 때문에, 정점이 모여서 이루어진 토폴로지 자체에 대한 조작은 불가능하다.
  * 노멀값은 삼각형 토폴로지의 중심이 되는 노멀이 아닌, 잠재적으로 삼각형을 이룰 수 있는 각 정점의 노멀을 사용한다.
  * 독립된 정점만을 가지고 쉐이더를 수행하기 때문에, 매우 효율적으로 병렬화해서 수행될 수 있다.
* **Input Assembler** (in D3D) 는 *VS* 이전에 일어나는 고정된 스테이지로, 버퍼와 버퍼 서술자에 따라 각각의 메모리 배열을 가지고 와서 **하나의 정점으로 취합** 해주는 스테이지이다.
  * Input Assembler 는 **Instancing** 을 가능하게 할 수 있다.
    자세한 설명은 `Section 18.4.2` 에서 설명한다고 한다.
  * Input Assembler 스테이지는 Logical 한 쉐이더 스테이지 모델에서 존재하고, Physical 한 쉐이더 파이프라인 구조에서는 존재하지 않을 수 있다. 논리적인 파이프라인에서와는 달리 물리적인 파이프라인에서는 *VS* 자체에서 정점을 취합해서 입력하게 할 수도 있다.
* *VS* 는 대개 다음과 같은 효과를 구현하고자 할 때 유용하게 사용될 수 있다.
  * Object Genration
  * Skinned Animation
  * Procedural Deformations `[802, 943]`
  * Terrain Height Field applying `[40, 1227]`

## 6. The Tessellation Stage

* Tessellation Stage (*TS*) 는 비교적 최근에 추가된 쉐이더 스테이지로, 기존의 **곡면 메쉬를 세분화하여 CPU 의 영향을 받지 않고 보다 더 부드러운 메쉬를 만들어주는 스테이지이다.**

  * D3D11, OpenGL 4.0, OpenGL ES 3.2 부터 추가되었음.

* *TS* 는 3 개의 스테이지로 구성된다. D3D11 에서는...

  1. **Hull Shader** : VS 에서 가져온 정점 패치를 입력으로 받아, 뒷 고정 스테이지인 *Tessellator* 에 테셀레이션의 강도(**Tessellation Factor**)를 정하게 하거나, Discard 하게 한다. OpenGL 에서는 *Tessellation Control Shader* 라고 함.
  2. **Tessellator** : *HS* 에서의 테셀레이션 정도를 사용해서 패치의 세분화된 토폴로지의 컨트롤 포인트 간의 세부 토폴로지를 만드는 고정된 테셀레이션을 수행한다. 이 스테이지에 의해서 새로운 정점들이 생성된다.
  3. **Domain Shader** : 입력으로 *HS* 에서의 컨트롤 포인트들을 받아서, 각각의 정점에 대한 출력을 결정한다. 이 프로그래밍 가능한 스테이지의 입출력은 *VS* 와 비슷한데, 즉 입력으로 테셀레이터에서 생성된 정점을 가지고, 작업을 한 뒤에 가공된 정점을 출력한다. OpenGL 에서는 *Tessellation Evaluation Shader* 라고 한다.

* **Tessellation Factor** 는 두 가지 레벨로 나뉜다.

  1. **Inner Edge** : 패치 (Patch) 의 각 삼각형에서 얼마만큼의 (재귀적인?) 삼각형 분화가 일어날 것인가를 결정.
  2. **Outer Edge** : 패치의 각 삼각형의 외부 간선이 몇 개로 분리될 것인가를 결정. 만약 $$ 0 $$ 인 경우에는 해당 삼각형을 Discard 한다.
     * 자세한 설명은 `Section 17.6` 에 있다고 한다.

* 테셀레이션에 의해 생성되는 각 정점은 **Barycentric Coordinate (삼각형의 3개의 정점의 중심을 기준으로 한 좌표계)** 를 사용해서 생성된다. 이를 사용해서 어떻게 정점을 생성하는 가는 `Section 22.8` 에서 자세히 다룬다.

  > http://mathworld.wolfram.com/BarycentricCoordinates.html
  > Barycentric Coordinate 는 대개 삼각형 (Simplex) 의 3 개의 정점 $$ (1, 0, 0), (0, 1, 0), (0, 0, 1) $$ (아핀 공간) 에 대해,
  > $$
  > \mathbf{p} = \sum_{i = 1}^{3}a_iv_i \\
  > \sum_{i = 1}^{3}a_i = 1
  > $$
  > 인 삼각형의 무게 중심 $$ \mathbf{p} $$ 을 기준으로 하는 좌표계이다.

## 7. The Geometry Shader

* Geometry Shader 는 **토폴로지 (Primitive) 를 다른 토폴로지로 변환시킬 수 있다.** 이는 이전 스테이지인 *HS 와 DS* 가 추가적인 면을 만들어서 부드러운 면을 만들게끔 하는 특화된 것과는 달리 보다 범용적이다. 예를 들어서, 삼각형을 선으로 변환시킬 수 있고, 혹은 삼각형을 삼각형들로 복제해서 유사 복셀을 구현할 수도 있다.
* 지오메트리 쉐이더를 사용한 예를 설명한 것으로는 `Section 10.4.3`
  혹은 쉐도우 맵핑 시 정점 정보를 각각의 쉐도우 맵에 렌더링하게 하는 것,
  또는 한번의 콜로 큐브맵의 각 면에 렌더링하게 하는 등의 테크닉이 있다.
* *GS* 는 변형된 프리미티브를 출력할 때, 인풋된 프리미티브의 어셈블리 순서대로 출력하도록 보장이 된다. 이는 곧 *GS* 을 씀으로써 전반적인 파이프라인의 성능에 영향을 미칠 수 있다는 것이 된다. *GS* 가 각기 다른 스레드에서 동작해서 아웃풋이 된다고 하면, 변형된 프리미티브를 파이프라인의 어셈블리 스코프에 넣기 위해 결과가 따로 보관되어야 하며, 재정렬되어야 한다. `[175, 530]`
  * *GS* 는 VS 나 PS 와는 다르게 많이 쓰이지 않으며, 이를 아예 배제하는 파이프라인도 존재한다.

### 7.1 Stream Output (Transform Feedback)

>  https://www.khronos.org/opengl/wiki/Transform_Feedback

* **Stream Output (Transform Feedback)** 은 레스터라이제이션 바로 이전의 단계에서 변형된 정점들의 정보를 일렬화된 버퍼를 통해 파이프라인 바깥으로 CPU 에게 출력할 수 있도록 한다.
  * 이 스테이지는 입력된 정점과 출력되는 정점의 순서들이 보장된다.
* 이 스테이지는 옵셔널하면서, 쓴다고 하더라도 레스터라이제이션 단계로도 가공된 정보들이 전달된다.
* *SO (TF)* 는 사양 상, 단정도 부동소수점의 형태로만 버퍼 출력이 가능하며, 버퍼 출력 시 정점의 순서에 따라서 버퍼에 정보가 입력되어야 하기 때문에 파이프라인의 성능에 영향을 미칠 수 있다.

## 8. The Pixel Shader

* *PS* 이전에, 래스터라이제이션 단계에서는 다음과 같은 작업을 한다.
  1. 정점들을 순서대로 순회해서, 스크린 상에 해당 정점들이 픽셀을 덮는 삼각형을 이루는가를 검사한다.
  2. 삼각형이 픽셀을 얼마나 덮는지를 계산한다. `Section 5.4.2`
  3. 고정 함수 상태에 의해서 삼각형 렌더링의 순서가 시계 방향 / 반시계 등등일 경우에 컬링을 할지 결정한다.
  4. 덮은 삼각형들에 대해, 각 정점의 출력의 값들을 **정해진 상태에 따라** 보간을 실시해서 Fragment 에 보간된 값들을 결정시킨다.
* 래스터라이제이션 단계에서, **각 픽셀을 덮은 삼각형 군(群)을** ***Fragment*** 라고 한다.
* *PS* 는 discard 기능을 가지고 있다. 또한 *PS* 는 렌더 타겟을 여러 개 지정하여, 컬러의 값을 각각의 렌더 타겟에 동시에 지정하게 할 수 있다. 이를 통해 Deferred Rendering 이 가능해짐.
  * 자세한 것은 `Section 20.1.` 에서 확인 가능.
* *PS* 는 `dv/dy` 혹은 `dv/dx` 와 같은,  2 x 2 의 블럭 **Quad** 에서 삼각형이 각 픽셀을 차지하는 양이 얼마나 변하는가를 확인할 수 있도록 API 을 제공해주고 있다. 이를 사용해서 텍스쳐 필터링으로 각종 효과를 덤으로 구현할 수 있을 것이다. 다만 이 경우에는 동적인 분기를 만들지 않도록 해야한다. 왜냐하면 `dv/dy` 와 `dv/dx` 는 다른 프래그먼트의 정보를 간접적으로 접근하기 때문에, 분기를 만들면 값이 무용지물된다고 보기 때문.

## 9. The Merging Stage

* 해당 스테이지를 D3D 에서는 *Output Merger*, 
  OpenGL 에서는 *per-sample Operation* 이라고 한다.

* 이 스테이지에서...

  1. 스텐실 테스트
  2. Z-테스트
  3. 블렌딩

  이 수행된다.

* Z-테스트의 경우, 마지막 스테이지에서 수행하기에는 씬이 복잡해지면 거의 대다수의 프래그먼트가 discard 되기 때문에, 성능 저하를 막기 위해 *PS* 에서 z값을 변경하는 코드가 들어있지 않으면, **early-z test** 을 래스터라이제이션 단계 이전에 수행하게 된다.

  * `Section 23.7` 과 `Section 18.4.5` 에서 이와 같은 성능 최적화를 좀 더 자세히 다룬다.

## 10. The Compute Shader

* GPGPU 와 같이, GPU Computing 을 사용해서 GPU 에 최적화된 연산을 수행하는 스테이지. 이 스테이지는 일반 GPU 렌더링 파이프라인에 묶이지 않으며, VRAM 에 존재하는 다양한 버퍼를 GPU 단에서 직접 조작하거나 접근하는 것이 가능하다.

* 이를 적극 활용해서, GPU 파티클 시스템, 메쉬 프로세싱, 얼굴 애니메이션, 컬링, 이미지 필터링, DOF, 쉐도우 등과 같은 작업이 가능하다. 뿐만 아니라, 컴퓨터 쉐이더를 사용하면 일반 테셀레이션을 사용하는 것보다 더 효율적이라는 문서도 나와있다.

## A. 참고

[943] : https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=1&ved=2ahUKEwjH146GzfLhAhWEErwKHT7LCaIQFjAAegQIABAB&url=https%3A%2F%2Fdownload.nvidia.com%2Fdeveloper%2FGPU_Gems_2%2FGPU_Gems2_ch18.pdf&usg=AOvVaw30ncpLaAEuIryWjR-m5cVi

[1227] : https://dl.acm.org/citation.cfm?id=1281671

[530] : https://fgiesen.wordpress.com/2011/07/09/a-trip-through-the-graphics-pipeline-2011-index/

