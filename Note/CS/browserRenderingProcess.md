# 💻 Browser Rendering Process

이 문서를 읽는 사람들이라면 HTML로 다양한 요소를 표현하고, CSS로 각 돔을 꾸며주고, JS로 다이나믹한 페이지를 만들 수 있을 것이라고 생각한다.

하지만 이것만 알아서는 그냥 준비된 종이에 그림만 그릴 줄 아는 것이다. 실제로 종이는 어떤 질감에 어떤 물감을 사용하는지 알아야 더 실력이 늘듯이 이 다음으로는 '실제 브라우저가 어떻게 돌아가는지'를 알아야 한다.
<br />
<br />

## 🖥  Critical Rendering Path
디스플레이가 초당 화면을 그리는 것을 fps (frame per second)라는 표현을 사용한다. 디스플레이가 1초에 60번 바뀌는데 웹페이지가 1초에 10번 그려진다면, 버벅이는 현상이 발생할 것이고 이런 현상을 Jank라고 부른다.
웹페이지의 렌더링의 과정을 알고 최적화를 한다면 브라우저가 1초에 60번 렌더링이 될 수 있도록 변경해줄 수 있게 된다. 
> "참 쉽죠?"

최적화를 하려면 브라우저가 화면을 어떻게 그리는지 알아야 하는데, 화면을 그릴 때 사용하는 엔진은 크게 **Webkit**엔진 (Chrome, safari)와 **Gecko**엔진 (Firefox)이 있다. 브라우저는 이런 엔진을 사용해서 화면을 그리는 데 거치는 주요한 과정을 `Critical Rendering Path (주요 렌더링 경로)`라고 부른다.

[자세한 내용은 이곳을](https://developer.mozilla.org/ko/docs/Web/Performance/Critical_rendering_path)

<br />
<br />

### 그래서 간단하게 브라우저의 렌더링 과정을 간단히 설명하자면...

1. HTML으로 작성된 데이터를 파싱한다.
2. 파싱한 결과로 **DOM Tree**를 만든다. 
3. CSS파일 또한 읽어서 **CSSOM** (*CSS Object Model*)를 만든다.
4. DOM과 CSSOM트리를 조합해 **Render Tree** 를 구축한다.
5. **Layout / Reflow** : [뷰 포트](https://developer.mozilla.org/ko/docs/Glossary/Viewport)내에서 각 노드들의 정확한 위치와 크기를 계산한다.
6. 계산한 것들을 기반으로 화면에 그린다.


### 🎈 HTML 파싱
- HTML의 문법에 맞게 작성된 마크업을 DOM으로 만들기 위해 각각 파싱한다. 그 과정에서 이미지, 비디오를 만나면 추가로 요청을 보낸다.
- 파싱하는 도중 JS를 만나면 실행 될 때까지 파싱을 멈춘다.
### 🎈 DOM Tree 만드는 과정
- 읽어들인 HTML 데이터를 문자열로 바꾸고, 문자열을 다시 읽어서 HTML 토큰으로 변환한다.
- 해당 토큰들은 Start Tag와 End Tag로 바뀌게 된다.
- Start Tag와 End Tag 사이에 있는 것들은 다시 같은 과정을 거치며 자식 노드로 들어가며 노드로 변환하는 작업을 한다. 노드로 변경되며 트리모양이 되는데 이 과정이 끝나면 Dom Tree가 생성된다.
### 🎈 CSSOM 만드는 과정
- HTML을 파싱하는 도중에 CSS 링크나 style 관련 정의를 만나면 해당 파일을 받아오거나 데이터를 파싱한다. CSS 또한 같은 과정을 거쳐 Tree 형태로 만들어진다.
### 🎈 Render Tree (Gecko: Frame Tree) 만들기
- 위에서 생성된 DOMTree와 CSSOM 을 합쳐서 `Render Tree (Gecko에서는 Frame Tree)`를 만든다.
  - 크기와 위치정보 제외 나머지부분들을 계산(css 우선순위 등에 따라)하여 트리를 생성한다.
- 렌더 트리는 실제로 보이는 것들로만 이루어지기 때문에 `{display: none}` 등으로 이루어진 것들은 Tree에 있어도 Render Tree에는 없고, head 태그의 메타 내용들도 존재하지 않는다.
### 🎈 Layout (Reflow) / 배치(리플로우)
- 렌더트리가 만들어졌는데 렌더트리는 크기와 위치정보를 가지지 않는다. 그러므로 어디에 배치가 될지 계산하는 과정을 거친다. 
- `position`, `width`, `height` 등등 위치에 관련된 부분들을 계산한다.
- resize 등이 일어나면 Render Tree가 변경되지 않고 Layout 이후의 과정만 거친다.
- `position: fixed`, `position: absolute`, `float: left` 등 전체적인 레이아웃(flow)와 분리된 레이아웃은 따로 노드를 만들어 처리한다.
### 🎈 Paint
- Layout 과정을 거치면 처리 된 렌더트리를 기반으로 화면을 그려주는 단계.
- 이 이후 수정이 생기면 다시 Layout(배치/리플로)를 하고 Paint를 함. 이를 Repaint라고 한다.

---

## 💡 Reflow / Repaint를 줄이는 팁.
- 동일 요소의 스타일 변경은 가급적 한번에 처리한다
```Javascript
const changeDivStyle = () => {
  const sample = document.querySelector(".sample");
  // bad (x)
  sample.style.width = '100px';
  sample.style.height = '100px';
  // right (o)
  sample.style.cssText = 'width: 100px; height: 100px;';

  /**
   * bad 예제를 보면 각각 고쳐서 reflow가 2번 일어나게 된다.
   * 그러므로 cssText로 한번만 실행되도록 할 수 있다.
   */
}
 
changeDivStyle();
```

- 애니메이션이 들어간 노드는 `position:fixed` 등으로 전체 레이아웃과 분리한 후 진행한다. 
  - 애니메이션은 엄청난 Layout(reflow)비용이 들어간다. 이 때 전체 레이아웃과 분리해주면 애니메이션이 필요한 노드만 계산하는데, 이렇게 하지 않으면 모든 노드들을 재계산 해야하기 때문에 많은 비용이 들어 느려지게 된다.
  
<br />
<br />

# 참고
- https://d2.naver.com/helloworld/59361