# 7 :: Vertex Processing & Drawing Command

## 7.1 Vertex Processing

*OpenGL* 에서 첫번째로 프로그래밍이 가능한 스테이지는 ***Vertex shader*** 이다. (그 이전의 스테이지는 Vertex fetching 으로 쉐이더에 필요한 정보 등을 가져오는 스테이지이다) 버텍스 쉐이더의 기본 목적은 ***클립 영역의 버텍스 위치***를 *pipeline* 의 다음 스테이지로 가져가는 것이다. 물론 이 과정 역시 생략될 수도 있는데 대개는 그렇지 않다.

### 7.1.1 Vertex Shader input

*OpenGL* 은 많은 종류의 버텍스 속성을 가질 수 있으며, 각 속성은 속성마다 자체 포맷을 가질 수도 있고, 데이터 타입, 컴포넌트 등이 다를 수 있다. 또한 *OpenGL* 은 다른 버퍼 객체로부터 각 속성에 대한 데이터를 읽을 수도 있다.

사실 일반적인 작업은 `glVetexAttribPointer` 로 해결이 가능하지만, 좀 더 저수준의 함수도 지원하고 있다. (`glVertexAttribPointer` 는 밑의 3 함수가 하는 것을 알아서 처리해준다.)

* ***`glVertexAttribFormat(attrib_index, size, type, normalized, relative_offset)`***
  정점 배열 (Vertex 배열) 의 데이터를 구성한다.
* ***`glVertexAttribBinding(attrib_index, binding_index)`***
* ***`glBindVertexBuffer(binding_index, buffer, offset, stride)`***

이 함수들이 어떻게 동작하는 지를 알기 위해서 다음과 같은 코드를 사용해본다.

``` c++
#version 430 core
layout (location = 0) in vec4 position;
layout (location = 1) in vec3 normal;
layout (location = 2) in vec2 tex_coord;
layout (location = 4) in vec4 color;
layout (location = 5) in int material_id;
```

그리고 다음과 같은 C 언어 구조체를 사용하기로 한다.

``` c
typedef struct VERTEX_t {
    float position[4];
    float normal[3];
    float tex_coord[2];
    unsigned byte color[3];
    int material_id;
} VERTEX;
```

#### A. glVertexAttribFormat

*location* 이 0, 1, 2 인 버텍스 속성에 대해서 `glVertexAttribFormat()` 을 사용할 때는, `size` 을 4, 3, 2 로 하고 `type` 을 `GL_FLOAT` 으로 하면 될 것이다. 하지만 *location* 이 4 인 color 에 대해서는 형이 다르기 때문에 살짝 까다롭다. *vec4* 는 4 바이트를 가지지만, C 구조체에서의 color 는 3 바이트를 가진다. 요소의 개수와 데이터 타입이 다른데, *OpenGL* 은 **데이터를 변환해서 페칭으로 넘겨준다**

``` c++
glVertexAttribFormat(4, 3, GL_UNSIGNED_BYTE, GL_TRUE, offsetof(VERTEX, color));
```

이렇게 하면, unsigned byte 인 color 의 값들은 255 로 나눠져 $ [0, 1] $ 의 값을 가지도록 **정규화**되고, 나머지 하나는 정해진 기본 값을 가진채로 버텍스 속성에 넘긴다.

> 이 함수에 대한 자세한 설명은 밑의 주소에서 볼 수 있다.
> https://www.khronos.org/registry/OpenGL-Refpages/gl4/html/glVertexAttribFormat.xhtml

만약에 `type` 을 지정할 때, 합쳐진 데이터 타입 (`GL_UNSIGNED_INT_2_10_10_10_REV` , `GL_INT_2_10_10_10_REV`) 을 사용하게 된다면, size 는 4 혹은 특별한 값인 **`GL_BGRA`** 로 지정해야 한다. 이렇게 함으로써 자동으로 입력 데이터를 ***Swizzling*** 하여 입력 벡터의 요소를  $ r, g, b, a $ 순으로 만들 수 있다. 

그리고 마지막으로 쉐이더에 데이터를 넘겨줄 때, **정수형**으로 꼭 넘겨줘야 한다면 (일반 함수는 죄다 부동소수점 형으로 넘긴다) 변형 함수인 `glVertexAttribIFormat` 으로 넘겨줄 수 있다.

``` c++
glVertexAttribIFormat(5, 1, GL_INT, offsetof(VERTEX, material_id));
```

마지막 인자인 `relative_offset` is the offset, **measured in basic machine units of the first element relative to the start of the vertex buffer** binding this attribute fetches from.

#### B. glVertexAttribBinding

이렇게 Format 함수를 사용해서 각 속성 인덱스에 어떤 버퍼를 사용해서 데이터를 패칭하겠다고 말했으면, 다음으로 *OpenGL* 에 **어떤 버퍼를 사용해서 데이터를 읽을 지 말해주어야 한다.** *OpenGL* 은 하드웨어에서 제한하는 개수까지의 버퍼로부터 읽어서 그 데이터를 제공할 수 있으며, 일부 버텍스 속성은 버퍼 내의 공간을 공유하도록 할 수 있다. 또한 다른 속성은 다른 버퍼 객체에 있을 수도 있다. 또는 *UBO* 와 같이, **바인딩 포인트 추상 계층을 하나 더 집어넣어 독립적으로 쓸 수 있도록 할 수 있다.**

여기서는 바인딩 포인트에 연관시켜서 *OpenGL* 에 매핑을 시킬 것이다. 

``` c++
glVertexAttribBinding(0, 0);
glVertexAttribBinding(1, 0);
/*! location 2, 4, 5 에 대해서도 똑같이... */
```

> 이 함수에 대한 자세한 설명은 다음 주소를 참고하라.
> https://www.khronos.org/registry/OpenGL-Refpages/gl4/html/glVertexAttribBinding.xhtml

하지만 더 복잡하게 속성 인덱스 4 를 버퍼 1에, 그리고 5 를 버퍼 2에 바인딩 시킬려고 한다면 다음과 같이 쓸 수도 있다.

``` c++
glVertexAttribBinding(4, 1);
glVertexAttribBinding(5, 2);
```

#### C. glBindVertexBuffer

* `binding_index` 는 버퍼를 바인딩하고자 하는 버퍼 바인딩 포인트의 인덱스이다.
* `buffer` 은 바인딩할 버퍼 객체의 이름이다.
* `offset` 은 정점 데이터가 시작하는 버퍼 객체 상의 오프셋 (바이트 단위) 이다.
* `stride` 는 각 버텍스 데이터의 시작 위치간의 바이트 단위 거리이다. 만약 촘촘하게 버텍스에 비어진 공간이 전혀 없다면 `sizeof` 로도 충분하다. 하지만 위에서는 공간이 빌 수도 있기 때문에 그냥 `0` 으로 하면 *OpenGL* 이 공간을 알아낸다.

> 이 함수에 대해 자세한 정보는 다음 주소에서 알 수 있다.
> https://www.khronos.org/registry/OpenGL-Refpages/gl4/html/glBindVertexBuffer.xhtml

### 7.1.2 버텍스 쉐이더 출력

버텍스 쉐이더의 주요한 내장 변수는 다음과 같다.

``` c++
out gl_PerVertex {
    vec4 gl_Position;
    float gl_PointSize;
    float gl_ClipDistance[];
};
```

`gl_Position` 은 각 정점에 대해 **클립 공간에 위치할 값**을 나타낸다.
그리고 `gl_PointSize` 는 렌더링 될 (수도 있는) **점들의 크기**를 제어한다. (최종 프리미티브가 GL_POINT 가 아니라면 반영되지 않을 수도 있다)
마지막으로 `gl_ClipDistance[]` 는 **클리핑**을 위해 사용된다.

> 자세한 사항은 Chapter 7.4 에서 설명한다.

#### 가변 점 크기

* ***`glPointSize(value)`*** 위치에 상관없는 렌더링 점 고정 크기를 갱신한다.

하지만 위 함수와는 다르게 점 크기를 프로그래밍으로 **가변적으로** 설정할 수 있는 방법이 존재한다. 이를 위해서는 쉐이더 프로그램에서 원하는 점 지름값을 내장 변수 `gl_PointSize` 에 쓰면 된다. 그렇게 하기 위해서는 *OpenGL* 에게 **`gl_PointSize`** 을 뜯어 고치도록 요청하는 과정이 필요하다.

* ***`glEnable(GL_PROGRAM_POINT_SIZE)`***
  을 사용해서 *VS* 에서 `gl_PointSize` 에 임의 값을 (하드웨어가 지원하는 선까지) 설정할 수 있다.

## 7.2 드로잉 커맨드

지금까지는 `glDrawArrays()` 만을 사용해서 렌더링 명령을 호출했지만, 하지만 *OpenGL* 은 여러 드로잉 커맨드를 제공한다. 

### 7.2.1 인덱스된 드로잉 커맨드

#### A. `glDrawElements()`

*glDrawArrays()* 커맨드는 인덱스되지 않은 드로잉 커맨드이다. 즉, 버텍스가 **순서대로 처리**되며, 어떠한 버텍스 데이터도 단순히 버퍼에 나타나는 순서대로 *Vertex shader*에 페칭된다. 한편 ***Index화된*** Draw 콜은, **미리 정해준 인덱스의 배열**을 사용해서 버퍼의 값들을 읽어 *Fetching*해서 나간다.

* **Index 화된** 드로잉 커맨드를 사용하기 위해서는 버퍼를 생성해서 해당 버퍼를 `GL_ELEMENT_ARRAY_BUFFER` 타깃에 바인딩을 해야한다. 이 버퍼에는 그리고자 하는 정점들의 **인덱스**가 들어가 있다.

* 그리고, 인덱스를 받는 드로잉 함수 중 하나를 호출한다. 인덱스된 드로잉 함수는 중간에 **`ELEMENT`** 가 들어가 있다. 예를 들어서 다음과 같은 함수가 존재한다.

  ``` c++
  void glDrawElements(GLenum mode, GLsizei count, GLenum type, const GLvoid * indices);
  ```

  이 함수는 `glDrawArrays()` 과 동일한 의미를 가지지만, `GL_ELEMENT_ARRAY_BUFFER` 의 인덱스 버퍼를 사용해서 해당 VAO 와 쉐이더를 동작시킨다는 점에서 차이점을 둔다.

  * `type` 은 해당 인덱스 버퍼의 **값의 형식**을 나타낸다. 대개 `GL_UNSIGNED_INT` 가 쓰인다.

> 해당 두 함수는 *OpenGL* 이 지원하는 기능 중 일부 기능만을 지원한다. 일반적인 드로잉 함수는 다음 주소의 See Also 을 참고하라.
> https://www.khronos.org/registry/OpenGL-Refpages/gl4/html/glDrawArrays.xhtml

####  B. Base vertex

추가 인자를 취하는 `glDrawElements()` 의 첫 번째 고급 버전은, `glDrawElementsBaseVertex()` 이다. 이 함수는 다음과 같은 인자를 취한다.

> https://www.khronos.org/opengl/wiki/GLAPI/glDrawElementsBaseVertex

* ``` c++
  void glDrawElementsBaseVertex(GLenum mode, GLsizei count, GLenum type, GLvoid *indices, GLint basevertex);
  ```

  `glDrawElements` 와 같으나, 다른 점은 버텍스 속성들이 페칭되기 전에 **인덱스에 `basevertex` 을 더함**으로써 시작 위치의 오프셋을 지정하도록 할 수 있다. 따라서 `glDrawElements` 는 `basevertex` 가 $ 0 $ 인 버전이라고 할 수 있다.

#### C. 프리미티브 재시작을 통한 지오메트리 합치기

여러가지 연속된 삼각형을 그려야 할 때, ***Triangle soup*** 라는 것을 통해 렌더링 성능을 향상시키게 할 수도 있다. 각 삼각형에는 세 개의 버텍스가 필요하고, 연속된 삼각형을 그릴려면 $ 3N$ 개의 정점이 필요하다. 하지만 *soup* 을 사해서 *strip* 화 시키면, 첫 삼각형을 제외한 모든 삼각형은 이미 그려진 정점들에서 연결할 정점 하나만 있으면 된다. 따라서 이 경우 $ N + 2 $ 개가 필요하다. 

하지만 *strip* 기능을 무턱대고 사용하면, 적절하지 않은 모델에서는 성능이 떨어지거나 스트립 사용 시에 어플리케이션이 제대로 처리를 못할 때도 존재한다.

* ***Primitive restart*** 는 이를 해결한다. 프리미티브 재시작은 `GL_TRIANGLE_STRIP` `GL_TRIANGLE_FAN` `GL_LINE_STRIP` `GL_LINE_LOOP` 등의 지오메트리 타입에 적용할 수 있다. 이것을 사용해서, *OpenGL* 에게 **언제 하나의 스트립이 끝나고, 다른 스트립이 시작하는가**를 알려줄 수 있다. 

  * 또한, 지오메트리에서는 하나의 스트립이 종료되고 시작되는 위치를 지정하기 위해서 **규격간 미리 정한 값으로 마커를 설정**한다. 따라서, *OpenGL* 은 Element 배열에서 인덱스를 가져올 때 이 특별한 인덱스를 확인해서 스트립을 갱신한다.

  * 그렇게 스트립을 갱신하기 위해서는 다음 함수로 활성화 시켜야한다.

    ``` c++
    glEnable(GL_PRIMITIVE_RESTART);
    glDisable(GL_PRIMITIVE_RESTART);
    ```

  * 스트립을 끊을 특별한 인덱스 값은 다음 함수로 설정할 수 있다.

    ``` c++
    glPrimitiveRestartIndex(value);
    ```

    이 값은 기본으로는 $ 0 $ 으로 되어있기 때문에, *strip* 을 쓸 참이라면 해당 `type` 의 해당 최대 값을 사용하는 것도 좋은 방법이 될 것이다.

  > 자세한 설명은 다음 주소를 참고하라.
  > https://www.khronos.org/opengl/wiki/Vertex_Rendering#Primitive_Restart

### 7.2.2 Instancing

#### A. Instancing

동일한 객체를 여러 번 반복해서 그려야 할 때, `glDrawArrays` 나 `glDrawElements` 로 매번 그리는 것은 효율이 매우 안 좋을 수 있다. 예를 들어서, 수십만의 간단한 개별 잔디 잎을 화면에 그려내야 한다고 하면, 이를 각각 렌더링하면 초당 프레임이 소수점을 찍어버릴 수도 있다.

사실 *OpenGL* 은 렌더링만 하기 때문에 문제가 되지 않을지도 모르겠지만, `for` 문 등을 사용해서 드로우 콜을 누적시킨다면 메모리와 GPU 사이의 버스 대역폭이 터져나가 병목이 일어날지도 모른다. 따라서 *OpenGL* 은 이 문제를 해결하기 위해 ***Instancing*** 을 지원하다. 이를 사용해서 GPU에서 **동일한 지오메트리의 많은 복사본을 그리도록 요청**하게 할 수 있다.

다음과 같은 함수를 쓴다.

* ``` c++
  void glDrawArraysInstanced(GLenum mode, GLint first, GLsizei count, GLsizei primcount);
  ```

  ``` c++
  void glDrawElementsInstanced(GLenum mode, GLsizei count, GLenum type, const void* indices, GLsizei primcount);
  ```

  위 함수들은 기존 드로잉 콜과 다를 바 없으나, **`primcount`** 로 *OpenGL* 에게 몇 개 동일한 지오메트리를 그릴지 명령할 수 있다는 차이점이 있다. 만약 `primcount` 을 $ 1 $ 로 설정하면 기존 함수와 동일한 동작을 요청한다. 

  * 또한, *OpenGL* 은 기존 `basevertex` 인자를 더한 *Instancing* 오버로딩 함수를 제공하고 있다.
  * `basevertex` 가 있듯이, *Instacing* 버전의 함수에는 ***`baseInstance`*** 인자를 제공하는 함수도 있다. 이 함수의 경우에는 인스턴싱을 할 때 정점 속성의 덩어리 역시 넘겨주게 되는데 이 정점 속성을 몇 번부터 시작하게 할까를 결정한다.

> 자세한 것은 다음 주소를 참고하라.
> https://www.khronos.org/opengl/wiki/GLAPI/glDrawArraysInstancedBaseInstance

* 인스턴스 렌더링이 가능한 이유는, 쉐이더 프로그램의 내장 변수인 ***`gl_InstanceID`*** 가 있기 때문이다. 이 변수는 버텍스에 들어있는데, *Instancing* 을 할 때 버텍스의 첫 번째 복사본이 쉐이더에 페칭이 되면 *gl_InstanceID* 는 $ 0 $ 이고, 다음 것이 들어오면 값은 $ 1 $ 이 된다.

또한 인스턴싱과 위에서 말한 것들을 십분 활용해서 동일한 지오메트리를 다르게 보이게 할 수도 있다.

#### B. 잔디 출력

> Chaper 5 의 _722_grass.cc 을 참고한다.

#### C. 데이터 자동으로 얻기

##### `gl_InstanceID`

이 외에도, 렌더링할 인스턴스 개수와 동일한 길이의 속성 배열이 있을 때, 인덱스로 `gl_InstanceID` 등을 사용할 수 있다. 예를 들어서 텍스쳐의 *texel* 을 참조하거나, *Uniform array* 의 인덱스로도 유용하게 활용할 수 있다. 이렇게 해서 **배열을 "인스턴스 속성"** 으로 간주한다. 렌더링 할 각 인스턴스마다 **새로운 속성값을 사용할 수 있게 하는 것이다**.

보통 정점 속성 (Vertex Attribute) 는 Vertex 별로 읽어서 새로운 값을 쉐이더에 제공하는데, 이와는 다르게 *OpenGL* 이 인스턴스 당 한 번씩 **배열로부터 속성을 읽도록** 하기 위해서는 다음과 같은 함수를 사용한다.

* ``` c++
  void glVertexAttribDivisor(GLuint index, GLuint divisor);
  ```

  * `index` 는 일반 Vertex Attribute 의 인덱스를 말한다.
  * **`divisor`** 은 *non-zero* 일 경우, ***divisor*** 개의 새로운 인스턴스마다 새로운 데이터를 Fetching 한다. 하지만 *zero* 일 경우는 버텍스마다 새로운 값을 갖는다. 

  이를 사용해서 매 속성에 대해 다른 값을 Fetching 할 수 있도록 한다.

##### 예시

> Chapter 5 의 _723_instance.cc 을 참고할 것.

### 7.2.3 Indirect Draw

#### A. 함수 설명 

이전까지는 정점의 개수 및 인스턴스의 개수, 그리고 보정 인덱스 및 보정 인스턴스와 같은 것들을 일일히 알아서 적어넣어줘야 했다. 하지만 **각 드로우의 인자를 버퍼 객체에 저장** 할 수 있도록 하는 렌더 함수 역시 존재한다. 다만 어플리케이션이 *OpenGL* 의 렌더 커맨드를 호출할 때는 버퍼 객체의 어떤 임의 인자에 무슨 자료가 들어가 있는지 전혀 모른다.

*OpenGL* 에는 $ 4 $ 개의 간접 렌더 커맨드가 존재한다. 이 중 두 개는 직접 드로잉 커맨드와 동일하다. 다음과 같은 함수가 존재한다.

* ``` c++
  void glDrawArraysIndirect(GLenum mode, const void *indirect);
  void glDrawElementsIndirect(GLenum mode, GLenum type, const void *indirect);
  ```

  위 함수는 직접 인자 쓰는 버전에 빗댈 때, `glDrawArraysInstancedBaseInstance`  와 `glDrawElementsInstancedBaseVertexBaseInstance` 와 같다.

  * **`indirect`** 는 ***`GL_DRAWY_INDIRECT_BUFFER`*** 타깃에 바인딩된 버퍼 객체의 주소를 입력으로 받는다.

따라서, 위 간접 함수에 대한 `INDIRECT` 버퍼는 다음과 같으며, 직접 함수를 쓸 때와 비교하면...

``` c++
typedef  struct {
    uint  count;
    uint  primCount;
    uint  first;
    uint  baseInstance;
} DrawArraysIndirectCommand;
```

``` c++
const DrawArraysIndirectCommand *cmd = (const DrawArraysIndirectCommand *)indirect;
glDrawArraysInstancedBaseInstance(mode, cmd->first, cmd->count, cmd->primCount, cmd->baseInstance);
```

와 동일하다. `Indirect` 구조체는 버퍼의 덩어리이기 때문에, 지정한 메모리 공간 위치만 맞으면 아무 버퍼나 `GL_DRAW_INDIRECT_BUFFER` 가 될 수 있다. (물론 이 버퍼를 쓰기 전에, 버퍼를 생성해서 바인딩해서 데이터를 집어넣어줘야 한다)

그리고, 이 작업을 훨씬 더 쉽게 수행할 수 있는, ***구조체 배열에 대한 루프를 도는 멀티 버전***이 존재한다. 

> https://www.khronos.org/registry/OpenGL-Refpages/gl4/html/glMultiDrawArraysIndirect.xhtml
> https://www.khronos.org/registry/OpenGL-Refpages/gl4/html/glMultiDrawElementsIndirect.xhtml

* ``` c++
  void glMultiDrawArraysIndirect(GLenum mode,
    	const void *indirect,
    	GLsizei drawcount,
    	GLsizei stride);
  ```

  *version 4.3* 부터 추가된 함수로, 이 함수는 **여러 개의 `GL_DRAW_INDIRECT_BUFFER`** 가 존재하면 이 컨테이너 청크를 순회하면서 간접적으로 프리미티브를 그린다. 이 역시 `glDrawElements` 버전을 지원한다.

  * `drawcount` 는 **순회할 횟수**를 말한다.
  * `stride` 는 각 `GL_DRAW_INDIRECT_BUFFER` 의 **stride** 을 말한다. 만약 $ 0 $ 이면 구조체들 사이의 메모리 공간이 없다고 가정하고 렌더링을 실시한다.

따라서 위의 함수들을 사용하면, **많은 드로잉 호출에 대한 각각의 인자들을 버퍼 객체에 미리 로딩**하게 할 수 있고, *GPU* 에서 각각의 드로잉을 하는데 필요한 버퍼 객체가 어플리케이션 단에서 준비되서 전송될 때까지 기다릴 필요가 없어진다.

다음 예제는 `glMultiDrawArraysIndirect()` 을 사용하는 간단한 예이다.

``` c++
typedef struct {
    GLuint count; GLuint primCount; GLuint first; GLuint baseInstance;
} DrawArraysIndirectCommand;

DrawArraysIndirectCommand draws[] = {
    { 42, 1, 0, 0 }, { 192, 1, 327, 0 }, { 99, 1, 901, 0 }
};

GLuint buffer;
glGenBuffers(1, &buffer);
glBindBuffer(GL_DRAW_INDIRECT_BUFFER, buffer);
glBufferData(GL_DRAW_INDIRECT_BUFFER, sizeof(draws), draws, GL_STATIC_DRAW);
glMultiDrawArraysIndirect(GL_TRIANGLES, buffer, sizeof(draws) / sizeof(draws[0]), 0);
```

#### B. 예시

> Chapter 7 / _723.indirect.cc 을 참고할 것.

## 7.3 변환된 버텍스 저장하기

#### A. [Transform feedback](https://www.khronos.org/opengl/wiki/Transform_Feedback)

**Transform Feedback** is the process of capturing [Primitives](https://www.khronos.org/opengl/wiki/Primitive) generated by the [Vertex Processing](https://www.khronos.org/opengl/wiki/Vertex_Processing) step(s), recording data from those primitives into [Buffer Objects](https://www.khronos.org/opengl/wiki/Buffer_Objects). This allows one to preserve the post-transform rendering state of an object and resubmit this data multiple times.

*Transform feedback* 은 프론트엔드의 실질적인 마지막 스테이지이며, 정점 처리 과정에서 생성된 프리미티브를 **캡쳐해서 다른 VBO (버퍼 오브젝트)** 에 기록할 수 있다. 이 단계는 프로그래밍 할 수 있지는 않지만 많은 설정이 가능하며 `gl` 로 시작하는 함수로 설정을 변경할 수 있다.

*Transform feedback* 에 기록되는 최종 프리미티브는 프론트엔드에서 프로그래밍 가능한 부분에서 최종 출력되는 프리미티브가 기록이 된다. 예를 들어서, *Geometry Shader* 가 없고, *TCS, TES* 가 없으면 *Vertex Shader* 에서 생성되서 *Primitive Assembler* 에서 조합된 프리미티브가 기록이 된다. 이 때 우선적으로 프리미티브의 정보가 기록이 되는 버퍼를 ***Transform Feedback Buffer*** 이라고 한다.

***TFB*** 에 최종 프리미티브가 들어가게 되면, `glGetBufferSubData()` 혹은 `glMapBufferRange(target, start, byte_size, flag)` 을 사용해서 어플리케이션 단에서 직접 읽게 할 수도 있다. 또한 이어지는 드로잉 커맨드의 원본 정점 데이터로 사용될 수 있다.

### 7.3.1 Transform Feedback 사용하기

#### A. Varying

> Attribute 을 `in` `out` 으로 쓰기 전에, `attirbute` `varying` 으로 쓰기도 하는데, 그 것의 의미와 같은 듯...

프론트엔드의 각각의 스테이지로부터의 출력을 **varying** 이라고 한다.

#### B. Functions

OpenGL 의 변환 피드백에 어떤 것을 기록할지 알리는 함수는 `glTransformFeedbackVarying()` 을 사용한다.

* [`glTransformFeedbackVarying(program, count, const GLchar* const* varying, bufferMode)`](https://www.khronos.org/registry/OpenGL-Refpages/gl4/html/glTransformFeedbackVaryings.xhtml) 
  * *program* : 쉐이더 프로그램의 ID
  * *count* : The number of varying variables used for transform feedback. 즉, *변환 피드백* 의 버퍼에 저장할 `varying` 쉐이더 변수들의 개수
  * *varying* : 쉐이더에서 찾을 `varying` 쉐이더 변수의 이름'들'
  * *bufferMode* : `varying` 을 변환 피드백 버퍼에 기록할 때 사용하는 옵션 플래그.
    * `GL_INTERLEAVED_ATTRIBS` : 여러 베어링은 **단일 버퍼**에 연속적으로 기록될 것이다.
    * `GL_SEPARATE_ATTRIBS` : 각 베어링은 **각 버퍼**에 연속적으로 기록될 것이다.

만약 쉐이더 프로그램에서 마지막 스테이지가 정점 쉐이더일 때, 버텍스 쉐이더 코드는 출력 베어링을 선언할 수 있다.

``` c++
out vec4 vs_position_out;
out vec4 vs_color_out;
out vec3 vs_normal_out;
out vec3 vs_binormal_out;
varying vec3 vs_tangent_out;
```

베어링을 지정하려면 어플리케이션 쪽에서, 해당 베어링 변수를 바인딩할 수 있도록 이름을 정해야 한다.

``` c++
static const char* varying_names[] = {
    "vs_position_out",
    "vs_color_out",
    "vs_normal_out",
    "vs_binormal_out",
    "vs_tangent_out"
};
const int num_varyings = sizeof(varying_names) / sizeof(varying_names[0]);
glTransformFeedbackVaryings(program, num_varyings, varying_names, GL_INTERLEAVED_ATTRIBS);
```

이 때 `glTransformFeedbackVarying` 은 쉐이더를 설정할 때 사용한다. 이 함수를 사용하고 난 뒤에는 ```glLinkProgram(program)``` 을 사용해서 프로그램 객체를 링크시켜줘야 한다.

만약 변환 피드백으로 기록된 베어링들을 변경하면, 프로그램 객체를 다시 링크시켜야 한다. (피드백 버퍼에 값을 변경하는게 아니라, 베어링 변수 그 자체를 변경시에는 객체를 다시 링크시키는 것은 당연하다.)

#### C. Implementation

*Transform feedback* 을 사용해서 변환 피드백에 *varying* 변수들의 값을 쓰기 전에, ***피드백 버퍼를 생성***해서 변환 피드백 버퍼 바인딩 포인트에 바인딩을 시켜야 한다. 물론 데이터를 쓰기 전에 버퍼에 공간이 할당되어야 한다.

``` c++
GLuint buffer;
glGenBuffer(1, &buffer);
glBindBuffer(GL_TRANSFORM_FEEDBACK_BUFFER, buffer);
glBufferData(GL_TRANSFORM_FEEDBACK_BUFFER, size, nullptr, GL_DYNAMIC_COPY);
```

> `GL_DYNAMIC_COPY` 는 데이터가 VRAM 에 상주한 뒤로도 자주 변하고, 매 갱신 중간에 사용될 수 있다는 것을 OpenGL 에게 알리는 옵션이다. 

이 후에, 어떤 버퍼에 변환 피드백 데이터가 바인딩이 될 것인지를 OpenGL 에게 알리기 위해서, **인덱스화된 Transform Feedback 바인딩 포인트** 중 하나에 저장해야 한다. 버퍼를 인덱스된 바인딩 포인트에 바인딩하기 위해서는 다음과 같은 함수를 호출한다.

* [**`glBindBufferBase(GL_TRANSFORM_FEEDBACK_BUFFER, index, buffer)`**](https://www.khronos.org/registry/OpenGL-Refpages/gl4/html/glBindBufferBase.xhtml)

  * *index* : GL_TRANSFORM_FEEDBACK_BUFFER 의 부수 바인딩 포인트이다. 이 때 index 값은 `glTransformFeedbackVarying` 함수에서 설정한 인덱스 값 $ N $ 의  $ N - 1 $ 까지여야 하고, zero based 이다.

* `glBindBufferRange(target, index, buffer, offset, size)` 

  일반 BaseBufferBase 에서 좀 더 기능이 확장된 버전으로, 이 함수를 사용해서 *Transform Feedback* 을 사용하면 바인딩할 버퍼의 일부를 인덱스된 바인딩 포인트에 바인딩 할  수 있다. 이를 활용하면, *Transform Feedback* 의 모드가 `GL_SEPARATE_ATTRIBS` 일 때에도 단일 버퍼의 각 일부분에 출력할 `varying` 들의 값을 저장할 수 있게 된다.

이 때, *변환 피드백 의 모드가 SEPARATE 일 때 해당 *TransformFeedback* 의 바인딩 포인트가 최대 몇 개 까지 허용하는 지를 알아보기 위해서는 다음과 같은 함수를 사용한다.

``` c++
glGetIntegerv(GL_MAX_TRANFORM_FEEDBACK_SEPARATE_COMPONENTS);
```

모드가 INTERLEAVED 일 때는 베어링 개수의 제한은 없지만, 버퍼에 쓸 수 있는 요소의 개수의 값은 제한이 있다. 예를 들면, `vec4` 보다는 `vec3` 베어링을 더 많이 쓸 수 있다. 이 제한은 그래픽스 하드웨어에 의존적이기 때문에 위와 같이 다음 함수를 사용해서 쓸 수 있다.

``` c++
glGetIntegerv(GL_MAX_TRANSFORM_FEEDBACK_INTERLEAVED_COMPONENTS);
```

>이 외에도...
>
>필요하다면 *TFB* 에 저장된 출력 구조체 사이에 빈 공간을 일부러 추가하게 할 수도 있다.
>이렇게 하려면, ***가상 베어링 이름*** 인, `gl_SkipCompontent1` `2` `3` `4` 를 사용해서 `glTransformFeedbackVarying()` 에서 같이 넘겨줘야 한다. 위 가상 이름은 버퍼의 타입에 대해 1~4개 요소의 빈 공간을 해당 베어링 사이에 만들어 준다.
>
>또한, 일련의 베어링 출력을 하나의 버퍼에 INTERLEAVED 로 쓰면서, 동시에 **일부 속성을 다른 버퍼**에 쓰게 하는 것도 가능하다. 이 경우에도 ***가상 베어링 이름*** 인 `gl_NextBuffer` 을 제공해주고 있으며, 이를 사용해서 현재 *TFB Binding point + 1* 에 바인딩된 버퍼에 값을 쓰게끔 한다.
>
>물론 이 경우에는, 변환 피드백 자체가 변경이 되므로, `glLinkProgram()` 을 호출해서 링크를 해줘야 한다.

### 7.3.2 Transform Feedback 시작, 일시 정지, 끝내기

#### A. Functions

변환 피드백을 설정하고, 결과를 담을 버퍼를 피드백 바인딩 포인트에 바인딩을 시킨 후에, 변환 피드백 모드는 다음 함수로 활성화 시킬 수 있다.

``` c++
void glBeginTransformFeedback(GLenum primitiveMode);
```

> https://www.khronos.org/registry/OpenGL-Refpages/gl4/html/glBeginTransformFeedback.xhtml

위 함수를 호출해서, 마지막 프론트엔드에서 출력 베어링이 변환 피드백 버퍼에 쓰여질 것이다. 물론 이 뒤에 `glDrawArrays()` 을 사용해서 드로잉 함수를 호출하여 버퍼에 베어링 변수들을 저장해야 한다. 이 때 중요한 것은, **변환 피드백에서 사용할 프리미티브와, 드로우 콜 시에 출력될 프리미티브와 같아야** 한다. *Geometry Shader* 가 있는 경우에는 `GL_PATCHES` 로, 그리고 쉐이더에서는 *TF* 와 동일한 프리미티브를 출력할 수 있어야 한다.

임시적으로 변환 피드백을 멈추고 싶을 경우에는 다음 함수를 호출하고, 일시정지인 상태에서 다시 변환 피드백 버퍼에 `varying` 을 쓰고자 할 때는 밑의 함수를 쓴다.

``` c++
void glPauseTransformFeedback();
void glResumeTransformFeedback();
```

요주의해야할 점은, *TF* 가 시작된 상태에서 변환 피드백 모드가 끝나거나 변환 피드백 버퍼에 할당된 공간을 다 쓸 때까지는 계속 *Transform Feedback Buffer* 에 `varying` 이 기록된다.

변환 피드백 모드를 저장할려면 다음을 호출한다.

``` c++
void glEndTransformFeedback();
```

> 주의해야 할 것은, 변환 피드백 모드가 시작되고 끝나는 도중에는 변환 피드백의 버퍼 크기를 조정하거나, 바인딩을 변경하거나 재할당하는 것은 불가능하다.

### 7.3.3 Backend 비활성화 하기

#### A. Functions

*Transform Feedback* 은 glsl 의 파이프라인 중에서 Frontend 만을 활용하고 Backend 을 쓰지 않는 경우가 많다. 따라서 쉐이더 단계에서 **그리지 않고자 하면** 다음 함수를 호출한다.

``` c++
glEnable(GL_RASTERIZER_DISCARD);
```

이렇게 하면 Frontend 뒤에 프리미티브 처리를 하지 않는다. 이 기믹은 *TF* 에서 유용하다. 만약 래스터라이제이션을 키려면 다음과 같이 쓴다.

``` c++
glDisable(GL_RASTERIZER_DISCARD);
```

#### 7.3.4 변환 피드백 예제

> Chapter7/_734_transformfeedback.cc 을 참고한다.

#### LoadShaders()

```c++
GLuint vertex_shader { sb6::shader::load(k_vs_update_path, GL_VERTEX_SHADER) };
glAttachShader(m_update_program, vertex_shader);
const char* transform_feedback_varyings[] { "tf_position_mass", "tf_velocity" };
glTransformFeedbackVaryings(
    m_update_program, 2, transform_feedback_varyings, GL_SEPARATE_ATTRIBS);
glLinkProgram(m_update_program);
glDeleteShader(vertex_shader);
```
*Transform feedback* 을 사용할 전용 쉐이더를 만든다. 이 쉐이더는 *Rasterization* 이후의 처리는 하지 않을 예정이기 때문에 (`glEnable(GL_RASTERIZER_DISCARD)`) 정점 쉐이더만 컴파일해서 링킹한다.

#### MakeInitialPositionWithConnection()

```c++
glGenVertexArrays(2, m_vao.data());
glGenBuffers(5, m_vbo.data());

for (auto i = 0; i < 2; i++) {
    glBindVertexArray(m_vao[i]);

    glBindBuffer(GL_ARRAY_BUFFER, m_vbo[POSITION_A + i]);
    glBufferData(GL_ARRAY_BUFFER, 
        total_byte_size<vmath::vec4>, initial_positions.get(), GL_DYNAMIC_COPY);
    glVertexAttribPointer(0, 4, GL_FLOAT, GL_FALSE, 0, NULL);
    glEnableVertexAttribArray(0);

    glBindBuffer(GL_ARRAY_BUFFER, m_vbo[VELOCITY_A + i]);
    glBufferData(GL_ARRAY_BUFFER, 
        total_byte_size<vmath::vec3>, initial_velocities.get(), GL_DYNAMIC_COPY);
    glVertexAttribPointer(1, 3, GL_FLOAT, GL_FALSE, 0, NULL);
    glEnableVertexAttribArray(1);

    glBindBuffer(GL_ARRAY_BUFFER, m_vbo[CONNECTION]);
    glBufferData(GL_ARRAY_BUFFER, 
        total_byte_size<vmath::ivec4>, connection_vectors.get(), GL_STATIC_DRAW);
    glVertexAttribIPointer(2, 4, GL_INT, 0, NULL);
    glEnableVertexAttribArray(2);
}
```
Update shader 안에서 유사 물리 처리를 할 예정이며, *Iteration* 을 해가면서 (반복) 할 예정이기 때문에, 각 점의 이전 포지션 및 속도를 담을 버퍼와, 처리 이후의 포지션 및 속도를 담을 버퍼를 만들어준다. 이 때 버퍼의 어드레스는 VRAM 에 위치하기 때문에 `initial_poisition` 등이 설정 이후 지워질 힙 객체이어도 상관은 없다.

#### render(double dt)

```c++
// Render update shader program till frontend stages.
glUseProgram(m_update_program);
glEnable(GL_RASTERIZER_DISCARD);

for (auto i = m_iterations_per_frame; i != 0; --i) {
    glBindVertexArray(m_vao[m_iteration_index & 1]);
    glBindTexture(GL_TEXTURE_BUFFER, m_pos_tbo[m_iteration_index & 1]);
    m_iteration_index++;
    
    glBindBufferBase(GL_TRANSFORM_FEEDBACK_BUFFER, 
    	0, m_vbo[POSITION_A + (m_iteration_index & 1)]);
    glBindBufferBase(GL_TRANSFORM_FEEDBACK_BUFFER, 
    	1, m_vbo[VELOCITY_A + (m_iteration_index & 1)]);
    
    // Begin transfrom feedback buffer storing.
    glBeginTransformFeedback(GL_POINTS);
    glDrawArrays(GL_POINTS, 0, k_point_total);
    glEndTransformFeedback();
}
glDisable(GL_RASTERIZER_DISCARD);
```
``` c++
glGenTextures(2, m_pos_tbo.data());
glBindTexture(GL_TEXTURE_BUFFER, m_pos_tbo[0]);
glTexBuffer(GL_TEXTURE_BUFFER, GL_RGBA32F, m_vbo[POSITION_A]);
glBindTexture(GL_TEXTURE_BUFFER, m_pos_tbo[1]);
glTexBuffer(GL_TEXTURE_BUFFER, GL_RGBA32F, m_vbo[POSITION_B]);
```

Update 물리 시뮬레이션을 하는 렌더링을 한다. 이 때 TextureBufferObject 가 바인딩이 되어있는데, TextureBufferObject 는 `m_vbo[POSITION_A, B]` 을 바인딩하고 있으며 (즉, 동일 버퍼가 두 군데에 바인딩이 되어있음) 이는 쉐이더 내부에서 이웃 노드의 접근을 하는데 매우 중요하게 사용된다. (무작위로 접근할 수 있도록 한다.)

``` glsl
for (int i = 0; i < 4; i++) {
    if (connection[i] != -1) {
        // q is the position of the other vertex
        vec3 q = texelFetch(tex_position, connection[i]).xyz;
        vec3 d = q - p;
        float x = length(d);
        F += -k * (rest_length - x) * normalize(d);
        fixed_node = false;
    }
}
```

``` c++
if (m_draw_lines) {
    glLineWidth(2.f);
    glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, m_index_buffer);
    glDrawElements(GL_LINES, k_total_connection * 2, GL_UNSIGNED_INT, NULL);
}
```

물체의 움직임에 따라서 선이 알아서 움직이는 것은, `GL_LINES` 을 사용했기 때문에, 각 정점에서 정점까지 렌더링을 알아서 해주는 것일 뿐이며, 쉐이더는 `Render` 로 바뀌었으나 아직 바인딩된 VAO 및 TBO 가 있기 때문에 묘화 렌더링을 할 때도 업데이트 된 VAO 의 버퍼 객체를 어셈블해서 렌더링하는 것이다.

#### In book...

> ...VBO 외에도 두 개의 TBO 가 필요하다. 각 버퍼를 위치 VBO 로도 사용하고 동시에 TBO 로도 사용한다. 좀 이상하게 들릴지도 모르겠지만, OpenGL 에서는 전혀 문제가 없다. 결국 두 개의 다른 방식을 사용해서 동일한 버퍼로부터 읽어오는 것이다.  이 때 필요한 함수가 ***`glTexBuffer()`*** 이다.
>
> ``` c++
> glGenTextures(2, m_pos_tbo.data());
> glBindTexture(GL_TEXTURE_BUFFER, m_pos_tbo[0]);
> glTexBuffer(GL_TEXTURE_BUFFER, GL_RGBA32F, m_vbo[POSITION_A]);
> glBindTexture(GL_TEXTURE_BUFFER, m_pos_tbo[1]);
> glTexBuffer(GL_TEXTURE_BUFFER, GL_RGBA32F, m_vbo[POSITION_B]);
> ```

### 7.4 Clipping (클리핑)

#### A. Clipping

클리핑은 어떤 프리미티브가 완전히 또는 부분적으로 보이는지 결정하고, 뷰포트 (Viewport) 안에 들어가는 일련의 프리미티브를 구성하는 작업이다.

점에 대해서는 클리핑이 단순한데, 상태 변수가 뷰포트 안에 *있느냐*, 아니면 *없느냐* 이 두가지로 나뉘기 때문이다.

하지만 선에 대해서는 약간 문제가 복잡하다. 선이 뷰포트 경계 가장자리에 걸치는 경우가 있기 때문이다. 이 경우에는 *내부적으로* 트리거가 되는 ***가장자리 면에 대해서 선을 부분 경계까지만 클리핑해서*** 부분적으로 그릴 것이 있는가 확인한 후에, 최종적으로 클리핑을 실시한다.

삼각형의 클리핑에서 가장자리에 걸칠 경우에는, ***여러 작은 삼각형을 뷰포트 가장자리 볼륨에 맞도록*** 잘라서 클리핑시켜야 한다. 따라서 가장자리에 걸쳐서 일부가 클리핑된 삼각형의 경우, 내부적으로 삼각형이 클리핑 되는 각 가장자리에 대해, **정점 하나와 삼각형 하나가** 추가적으로 생기게 된다.

혹시나 이 삼각형 클리핑이 GPU 성능 저하를 가져올 수도 있기 때문에, 특정 GPU 는 **보호 띠** 라는 개념을 사용해서 현재 설정한 Viewport 보다 더 큰 Viewport 을 내부적으로 생성하여 클리핑을 막고 래스터라이저에서 버릴 부분을 버리는 방법을 택하기도 한다