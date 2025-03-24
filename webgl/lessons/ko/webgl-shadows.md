Title: WebGL 그림자
Description: 그림자를 계산하는 방법
TOC: 그림자


그림자를 그려봅시다!

## 사전지식

기본적인 그림자를 계산하는 것은 *그렇게까지* 어렵지는 않지만 많은 배경지식을 필요로 합니다.
이 글을 이해하기 위해서는 아래 내용에 대해 알고 계셔야 합니다.

* [직교 투영](webgl-3d-orthographic.html)
* [원근 투영](webgl-3d-perspective.html)
* [스포트라이트 효과](webgl-3d-lighting-spot.html)
* [텍스처](webgl-3d-textures.html)
* [텍스처 렌더링](webgl-render-to-texture.html)
* [투영 매핑](webgl-planar-projection-mapping.html)
* [카메라 시각화](webgl-visualizing-the-camera.html)

그러니 위 글들을 아직 읽지 않으셨다면 먼저 읽고 오십시오.

무엇보다, 이 글은 여러분이 [유틸리티 함수에 대한 글](webgl-less-code-more-fun.html)을 이미 읽으셨다고 가정하고 있습니다.
이 글의 예제에서는 그 라이브러리를 사용해서 코드를 단순화 하고 있습니다.
여러분이 버퍼, 정점 배열과 속성이 뭔지 모르겠다거나 `webglUtils.setUniforms`같은 코드를 봤는데 유니폼을 설정하는 게 무슨 뜻인지 모르시겠다면, 더 앞으로 돌아가서 [기초 부분부터 읽고 오십시오](webgl-fundamentals.html).

우선 그림자를 그리는 데는 여러가지 방법이 있습니다.
각 방법들은 장단점을 가지고 있습니다.
가장 흔히 사용되는 방법은 그림자 맵을 사용해서 그림자를 그리는 것입니다.

섀도우 맵은 사전지식에서 언급한 모든 기술을 사용하여 동작합니다.

[투영 매핑과 관련된 글](webgl-planar-projection-mapping.html)에서 이미지를 물체에 투영하는 방법을 알아 보았습니다.

{{{example url="../webgl-planar-projection-with-projection-matrix.html"}}}

이미지는 장면에 있는 물체에 직접 그려진 것이 아니라, 물체가 렌더링 될 때 각 픽셀에 대해 투영된 텍스처 범위 내에 있는지를 확인하고 범위 내에 있다면 투영된 텍스처로부터 적절한 색상을 샘플링하는 방식이었습니다.
범위 밖이라면 물체에 매핑된 다른 텍스처로부터 텍스처 좌표를 기반으로 색상을 샘플링하였습니다.

만일 투영된 텍스처가 조명의 시점에서 얻어진 깊이 데이터라면 어떻게 될까요?
다시 말해 위 예제에서 조명이 절두체의 끝부분에 존재하는 것처럼 가정하고 투영된 텍스처가 그 조명 위치에서의 깊이값을 가지고 있는겁니다.
그러면 구는 조명에 가까운 깊이값을 가질거고 평면은 더 먼 깊이값을 가질겁니다.

<div class="webgl_center"><img class="noinvertdark" src="resources/depth-map-generation.svg" style="width: 600px;"></div>

그런 데이터를 확보할 수 있다면 렌더링할 색상을 결정할 때 투영된 텍스처로부터 깊이값을 얻어올 수 있고, 그리려는 픽셀이 그 깊이값보다 조명과 더 먼지 더 가까운지 알 수 있습니다.
만일 더 멀다면 다른 더 조명과 가까운 물체가 있다는 뜻입니다.
다시 말해 무언가가 조명을 가리고 있어서 픽셀이 그림자 영역에 있다는 겁니다.

<div class="webgl_center"><img class="noinvertdark" src="resources/projected-depth-texture.svg" style="width: 600px;"></div>

보시면 깊이 텍스처가 조명 시점에서의 절두체 조명 공간을 통해 투영되고 있습니다.
우리가 바닥면 픽셀을 그릴 때 그 픽셀의 조명으로부터의 깊이를 계산합니다. (위 그림에서 0.3)
그리고 나서 깊이 지도 텍스처에서는 이와 대응하는 깊이값을 찾아봅니다.
조명 시점에서 텍스처에 저장된 깊이값은 0.1인데, 구가 있기 때문입니다.
0.1 &lt; 0.3이므로 그 바닥면 픽셀은 그림자 범위에 있다는 것을 알 수 있습니다.

먼저 섀도우 맵을 그려봅시다.
[평면 투영 매핑에 관한 글](webgl-planar-projection-mapping.html)의 마지막 예제를 가져오지만 텍스처를 로딩하는 대신에 [텍스처에 렌더링](webgl-render-to-texture.html)할 겁니다.
[해당 글](webgl-render-to-texture.html)에서는 깊이 렌더 버퍼를 사용했습니다.
이는 픽셀 정렬을 돕는 깊이 버퍼를 제공했지만 깊이 렌더 버퍼를 텍스처로 사용할 수는 없습니다.
다행히도 `WEBGL_depth_texture`라는 선택적 WebGL 확장이 있어 이를 활성화하여 깊이 텍스처를 제공할 수 있습니다.
깊이 텍스처를 사용하여 프레임 버퍼에 첨부한 다음 나중에 셰이더에 대한 입력으로 텍스처를 사용합니다.
다음은 확장이 있는지 확인하고 활성화하는 코드입니다.

```js
function main() {
  // WebGL Context 얻기
  /** @type {HTMLCanvasElement} */
  const canvas = document.querySelector('#canvas');
  const gl = canvas.getContext('webgl');
  if (!gl) {
    return;
  }

+  const ext = gl.getExtension('WEBGL_depth_texture');
+  if (!ext) {
+    return alert('need WEBGL_depth_texture');
+  }
```

이제 [텍스처 렌더링에 관한 글](webgl-render-to-texture.html)과 유사하게 텍스처를 생성한 다음 프레임 버퍼를 생성하고 `DEPTH_ATTACHMENT`로 프레임 버퍼에 텍스처를 첨부합니다.

```js
const depthTexture = gl.createTexture();
const depthTextureSize = 512;
gl.bindTexture(gl.TEXTURE_2D, depthTexture);
gl.texImage2D(
    gl.TEXTURE_2D,      // 대상
    0,                  // 밉 레벨
    gl.DEPTH_COMPONENT, // 내부 포맷
    depthTextureSize,   // 너비
    depthTextureSize,   // 높이
    0,                  // 테두리
    gl.DEPTH_COMPONENT, // 포맷
    gl.UNSIGNED_INT,    // 타입
    null);              // 데이터
gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MAG_FILTER, gl.NEAREST);
gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, gl.NEAREST);
gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_S, gl.CLAMP_TO_EDGE);
gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_T, gl.CLAMP_TO_EDGE);

const depthFramebuffer = gl.createFramebuffer();
gl.bindFramebuffer(gl.FRAMEBUFFER, depthFramebuffer);
gl.framebufferTexture2D(
    gl.FRAMEBUFFER,       // 대상
    gl.DEPTH_ATTACHMENT,  // 어태치먼트 포인트
    gl.TEXTURE_2D,        // 텍스처 대상
    depthTexture,         // 텍스처
    0);                   // 밉 레벨
```

[여러 가지 이유](#attachment-combinations)로 실제로 사용하지 않더라도 색상 텍스처를 생성하고 색상 attachment로 첨부해야 합니다.

```js
// 깊이 텍스처와 같은 크기로 색상 텍스처 생성
const unusedTexture = gl.createTexture();
gl.bindTexture(gl.TEXTURE_2D, unusedTexture);
gl.texImage2D(
    gl.TEXTURE_2D,
    0,
    gl.RGBA,
    depthTextureSize,
    depthTextureSize,
    0,
    gl.RGBA,
    gl.UNSIGNED_BYTE,
    null,
);
gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MAG_FILTER, gl.NEAREST);
gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, gl.NEAREST);
gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_S, gl.CLAMP_TO_EDGE);
gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_T, gl.CLAMP_TO_EDGE);

// 프레임 버퍼에 첨부
gl.framebufferTexture2D(
    gl.FRAMEBUFFER,        // 대상
    gl.COLOR_ATTACHMENT0,  // 어태치먼트 포인트
    gl.TEXTURE_2D,         // 텍스처 대상
    unusedTexture,         // 텍스처
    0);                    // 밉 레벨
```

이를 사용하기 위해서는 서로다른 셰이더를 사용해서 장면을 두번 이상 그릴 수 있어야 합니다.
한 번은 깊이 텍스처로 렌더링하기위한 간단한 셰이더를 사용해서, 한 번은 텍스처를 투영하는 현재 셰이더를 사용해서 그릴겁니다.

먼저 `drawScene`을 수정해서 우리가 렌더링을 수행하려는 프로그램을 전달할 수 있도록 합시다.

```js
-function drawScene(projectionMatrix, cameraMatrix, textureMatrix) {
+function drawScene(projectionMatrix, cameraMatrix, textureMatrix, programInfo) {
  // 카메라 행렬로 뷰 행렬을 만듭니다.
  const viewMatrix = m4.inverse(cameraMatrix);

-  gl.useProgram(textureProgramInfo.program);
+  gl.useProgram(programInfo.program);

  // 구체와 평면에 모두 사용되는 유니폼을 설정합니다.
  // 주의: 셰이더에 대응되는 유니폼이 없는 경우 무시됩니다.
-  webglUtils.setUniforms(textureProgramInfo, {
+  webglUtils.setUniforms(programInfo, {
    u_view: viewMatrix,
    u_projection: projectionMatrix,
*    u_textureMatrix: textureMatrix,
-    u_projectedTexture: imageTexture,
+    u_projectedTexture: depthTexture,
  });

  // ------ 구체 그리기 --------

  // 속성 설정
-  webglUtils.setBuffersAndAttributes(gl, textureProgramInfo, sphereBufferInfo);
+  webglUtils.setBuffersAndAttributes(gl, programInfo, sphereBufferInfo);

  // 구에 필요한 유니폼 설정
-  webglUtils.setUniforms(textureProgramInfo, sphereUniforms);
+  webglUtils.setUniforms(programInfo, sphereUniforms);

  // gl.drawArrays 또는 gl.drawElements 호출
  webglUtils.drawBufferInfo(gl, sphereBufferInfo);

  // ------ 평면 그리기 --------

  // 속성 설정
-  webglUtils.setBuffersAndAttributes(gl, textureProgramInfo, planeBufferInfo);
+  webglUtils.setBuffersAndAttributes(gl, programInfo, planeBufferInfo);

  // 위에서 계산한 유니폼 설정
-  webglUtils.setUniforms(textureProgramInfo, planeUniforms);
+  webglUtils.setUniforms(programInfo, planeUniforms);

  // gl.drawArrays 또는 gl.drawElements 호출
  webglUtils.drawBufferInfo(gl, planeBufferInfo);
}
```

이제 `drawScene`을 활용해 장면을 조명 시점에서 그리고, 그 후에 깊이 텍스처를 사용해 그립니다.

```js
function render() {
  webglUtils.resizeCanvasToDisplaySize(gl.canvas);

  gl.enable(gl.CULL_FACE);
  gl.enable(gl.DEPTH_TEST);

  // 조명 시점에서 먼저 그립니다.
-  const textureWorldMatrix = m4.lookAt(
+  const lightWorldMatrix = m4.lookAt(
      [settings.posX, settings.posY, settings.posZ],          // 위치
      [settings.targetX, settings.targetY, settings.targetZ], // 대상
      [0, 1, 0],                                              // 위쪽
  );
-  const textureProjectionMatrix = settings.perspective
+  const lightProjectionMatrix = settings.perspective
      ? m4.perspective(
          degToRad(settings.fieldOfView),
          settings.projWidth / settings.projHeight,
          0.5,  // 근거리
          10)   // 원거리
      : m4.orthographic(
          -settings.projWidth / 2,   // 왼쪽
           settings.projWidth / 2,   // 오른쪽
          -settings.projHeight / 2,  // 아래쪽
           settings.projHeight / 2,  // 위쪽
           0.5,                      // 근거리
           10);                      // 원거리

+  // 깊이 텍스처에 그립니다.
+  gl.bindFramebuffer(gl.FRAMEBUFFER, depthFramebuffer);
+  gl.viewport(0, 0, depthTextureSize, depthTextureSize);
+  gl.clear(gl.COLOR_BUFFER_BIT | gl.DEPTH_BUFFER_BIT);

-  drawScene(textureProjectionMatrix, textureWorldMatrix, m4.identity());
+  drawScene(lightProjectionMatrix, lightWorldMatrix, m4.identity(), colorProgramInfo);

+  // 이번에는 캔버스에 그리는데, 깊이 텍스처를 장면에 투영해서 그립니다.
+  gl.bindFramebuffer(gl.FRAMEBUFFER, null);
+  gl.viewport(0, 0, gl.canvas.width, gl.canvas.height);
+  gl.clear(gl.COLOR_BUFFER_BIT | gl.DEPTH_BUFFER_BIT);

  let textureMatrix = m4.identity();
  textureMatrix = m4.translate(textureMatrix, 0.5, 0.5, 0.5);
  textureMatrix = m4.scale(textureMatrix, 0.5, 0.5, 0.5);
-  textureMatrix = m4.multiply(textureMatrix, textureProjectionMatrix);
+  textureMatrix = m4.multiply(textureMatrix, lightProjectionMatrix);
  // 월드 행렬의 역행렬을 사용합니다.
  // 이렇게 하면 다른 위치 값들이 이 월드 공간에 상대적인 값이 됩니다.
  textureMatrix = m4.multiply(
      textureMatrix,
-      m4.inverse(textureWorldMatrix));
+      m4.inverse(lightWorldMatrix));

  // 투영 행렬 계산
  const aspect = gl.canvas.clientWidth / gl.canvas.clientHeight;
  const projectionMatrix =
      m4.perspective(fieldOfViewRadians, aspect, 1, 2000);

  // lookAt을 사용한 카메라 행렬 계산
  const cameraPosition = [settings.cameraX, settings.cameraY, 7];
  const target = [0, 0, 0];
  const up = [0, 1, 0];
  const cameraMatrix = m4.lookAt(cameraPosition, target, up);

-  drawScene(projectionMatrix, cameraMatrix, textureMatrix);
+  drawScene(projectionMatrix, cameraMatrix, textureMatrix, textureProgramInfo);
}
```

`textureWorldMatrix`를 `lightWorldMatrix`로, `textureProjectionMatrix`를 `lightProjectionMatrix`로 이름을 바꾼 것에 유의하세요.
전에는 텍스처를 임의의 공간으로 투영했지만 지금은 섀도우 맵을 조명에서 투영하고 있습니다.
계산 방법은 같지만 변수 이름을 바꾸는 것이 적절해 보입니다.

먼저 구와 평면을 절두체 라인을 그리기 위해 만든 색상 셰이더를 사용해 깊이 텍스처에 렌더링했습니다.
이 셰이더는 단색을 그리는 셰이더이고 특별히 다른 계산을 하고 있지 않은데
깊이 텍스처를 렌더링하는데는 이것이면 충분합니다.

이후에 장면을 캔버스에 다시 그리는데 전과 동일하게 텍스처를 장면에 투영해서 그립니다.
셰이더에서 깊이 텍스처를 참조할 때 `red` 값만 유효하기 때문에
`red`, `green`, `blue`에 대해 같은 값을 반복하여 할당합니다.

```glsl
void main() {
  vec3 projectedTexcoord = v_projectedTexcoord.xyz / v_projectedTexcoord.w;
  bool inRange =
      projectedTexcoord.x >= 0.0 &&
      projectedTexcoord.x <= 1.0 &&
      projectedTexcoord.y >= 0.0 &&
      projectedTexcoord.y <= 1.0;

-  vec4 projectedTexColor = texture2D(u_projectedTexture, projectedTexcoord.xy);
+  // 'r'채널에 깊이값이 저장되어 있습니다.
+  vec4 projectedTexColor = vec4(texture2D(u_projectedTexture, projectedTexcoord.xy).rrr, 1);
  vec4 texColor = texture2D(u_texture, v_texcoord) * u_colorMult;
  float projectedAmount = inRange ? 1.0 : 0.0;
  gl_FragColor = mix(texColor, projectedTexColor, projectedAmount);
}
```

이제 장면에 육면체를 추가해 봅시다.

```js
+const cubeBufferInfo = primitives.createCubeBufferInfo(
+    gl,
+    2,  // 크기
+);

...

+const cubeUniforms = {
+  u_colorMult: [0.5, 1, 0.5, 1],  // 밝은 초록색
+  u_color: [0, 0, 1, 1],
+  u_texture: checkerboardTexture,
+  u_world: m4.translation(3, 1, 0),
+};

...

function drawScene(projectionMatrix, cameraMatrix, textureMatrix, programInfo) {

    ...

+    // ------ 육면체를 그립니다. --------
+
+    // 필요한 속성을 설정합니다.
+    webglUtils.setBuffersAndAttributes(gl, programInfo, cubeBufferInfo);
+
+    // 방금 계산한 유니폼을 입력합니다.
+    webglUtils.setUniforms(programInfo, cubeUniforms);
+
+    // gl.drawArrays 또는 gl.drawElements를 호출합니다.
+    webglUtils.drawBufferInfo(gl, cubeBufferInfo);

...
```

세팅을 조금 바꿔 보죠. 카메라를 약간 이동하고 시야각을 넓혀서 투영되는 텍스처가 더 많은 부분을 덮도록 해 봅시다.

```js
const settings = {
-  cameraX: 2.5,
+  cameraX: 6,
  cameraY: 5,
  posX: 2.5,
  posY: 4.8,
  posZ: 4.3,
  targetX: 2.5,
  targetY: 0,
  targetZ: 3.5,
  projWidth: 1,
  projHeight: 1,
  perspective: true,
-  fieldOfView: 45,
+  fieldOfView: 120,
};
```

주의: 절두체를 보여주기 위해 라인을 그리는 코드는 `drawScene` 함수 밖으로 옮겼습니다.

{{{example url="../webgl-shadows-depth-texture.html"}}}

이미지를 로딩하는 대신 장면을 깊이 텍스처에 렌러딩한 결과를 사용한다는 것만 제외하면 위쪽의 예제와 완전히 동일합니다.
`cameraX`를 2.5로 `fieldOfView`를 45로 바꾸면 위쪽과 동일한 결과를 볼 수 있습니다.
로딩된 이미지 대신 우리의 깊이 텍스처가 투영되고 있다는 점만 제외하면요.

깊이값은 0.0에서 1.0사이의 값인데 절두체 내에서의 위치를 나타냅니다.
그러니 0.0(어두움)은 절두체의 뾰족한 점에 가깝고 1.0(밝음)은 반대쪽 끝에 가깝습니다.

이제 남은 것은 투영된 텍스처 색상과 텍스처 매핑 색상 사이의 선택을 하는 것이 아니라, 깊이 텍스처의 깊이 값에서 얻은 Z값을 사용해 그 값이 우리가 그리려는 픽셀에서 카메라까지의 거리보다 더 먼지 가까운지를 알아내는데 사용하는 것입니다.
깊이 텍스처의 값이 더 가까우면 무언가가 빛을 가리고 있는 것이고, 따라서 그 픽셀은 그림자 영역 안에 있습니다.

```glsl
void main() {
  vec3 projectedTexcoord = v_projectedTexcoord.xyz / v_projectedTexcoord.w;
+  float currentDepth = projectedTexcoord.z;

  bool inRange =
      projectedTexcoord.x >= 0.0 &&
      projectedTexcoord.x <= 1.0 &&
      projectedTexcoord.y >= 0.0 &&
      projectedTexcoord.y <= 1.0;

-  vec4 projectedTexColor = vec4(texture2D(u_projectedTexture, projectedTexcoord.xy).rrr, 1);
+  float projectedDepth = texture2D(u_projectedTexture, projectedTexcoord.xy).r;
+  float shadowLight = (inRange && projectedDepth <= currentDepth) ? 0.0 : 1.0;

  vec4 texColor = texture2D(u_texture, v_texcoord) * u_colorMult;
-  gl_FragColor = mix(texColor, projectedTexColor, projectedAmount);
+  gl_FragColor = vec4(texColor.rgb * shadowLight, texColor.a);
}
```

위에서 `projectedDepth`가 `currentDepth`보다 작으면 조명의 시점에서 더 가까운 물체가 있는 것이므로 그리려는 픽셀이 그림자 영역 안에 있는 것입니다.

실행해 보면 그림자가 나타납니다.

{{{example url="../webgl-shadows-basic.html" }}}

구의 그림자가 바닥면에 나타나는 것을 보니 되는 것 같기는 한데, 그림자가 없어야 하는 곳에 나타나는 이상한 패턴을 뭘까요?
이 패턴은 *섀도우 애크니*이라고 합니다.
깊이 텍스처에 저장된 깊이 데이터가 양자화되기 때문입니다.
이는 텍스처 자체가 픽셀의 그리드이기 때문이기도 하고, 조명의 시점으로 투영되어 생성되었으나 그 값을 카메라 시점에서 비교하고 있기 때문이기도 합니다.
다시말해 깊이 지도 격자의 값들이 카메라와 정렬되지 않아서 `currentDepth`를 계산할 때 `projectedDepth`보다 약간 작거나 큰 경우가 생기기 때문입니다.

바이어스를 더해 봅시다.

```glsl
...

+uniform float u_bias;

void main() {
  vec3 projectedTexcoord = v_projectedTexcoord.xyz / v_projectedTexcoord.w;
-  float currentDepth = projectedTexcoord.z;
+  float currentDepth = projectedTexcoord.z + u_bias;

  bool inRange =
      projectedTexcoord.x >= 0.0 &&
      projectedTexcoord.x <= 1.0 &&
      projectedTexcoord.y >= 0.0 &&
      projectedTexcoord.y <= 1.0;

  float projectedDepth = texture2D(u_projectedTexture, projectedTexcoord.xy).r;
  float shadowLight = (inRange && projectedDepth <= currentDepth) ? 0.0 : 1.0;

  vec4 texColor = texture2D(u_texture, v_texcoord) * u_colorMult;
  gl_FragColor = vec4(texColor.rgb * shadowLight, texColor.a);
}
```

값을 설정해 줍니다.

```js
const settings = {
  cameraX: 2.75,
  cameraY: 5,
  posX: 2.5,
  posY: 4.8,
  posZ: 4.3,
  targetX: 2.5,
  targetY: 0,
  targetZ: 3.5,
  projWidth: 1,
  projHeight: 1,
  perspective: true,
  fieldOfView: 120,
+  bias: -0.006,
};

...

function drawScene(projectionMatrix, cameraMatrix, textureMatrix, programInfo, /**/u_lightWorldMatrix) {
  // 카메라 행렬로 뷰 행렬을 만듭니다.
  const viewMatrix = m4.inverse(cameraMatrix);

  gl.useProgram(programInfo.program);

  // 구와 평면에 모두 사용되는 유니폼을 설정합니다.
  // 주의: 셰이더에 대응되는 유니폼이 없는 경우 무시됩니다.
  webglUtils.setUniforms(programInfo, {
    u_view: viewMatrix,
    u_projection: projectionMatrix,
+    u_bias: settings.bias,
    u_textureMatrix: textureMatrix,
    u_projectedTexture: depthTexture,
  });

  ...
```

{{{example url="../webgl-shadows-basic-w-bias.html"}}}

바이어스 값을 바꿔보면 패턴이 나타나는 위치와 시점에 영향을 주는 것을 알 수 있습니다.

코드를 완성하기 위해 [스포트라이트 효과](webgl-3d-lighting-spot.html)의 스포트라이트 계산 코드를 추가하도록 하겠습니다.

먼저 [이 글](webgl-3d-lighting-spot.html)의 정점 셰이더에서 필요한 부분을 가져다 붙입시다.

```glsl
attribute vec4 a_position;
attribute vec2 a_texcoord;
+attribute vec3 a_normal;

+uniform vec3 u_lightWorldPosition;
+uniform vec3 u_viewWorldPosition;

uniform mat4 u_projection;
uniform mat4 u_view;
uniform mat4 u_world;
uniform mat4 u_textureMatrix;

varying vec2 v_texcoord;
varying vec4 v_projectedTexcoord;
+varying vec3 v_normal;

+varying vec3 v_surfaceToLight;
+varying vec3 v_surfaceToView;

void main() {
  // 위치와 행렬을 곱합니다.
  vec4 worldPosition = u_world * a_position;

  gl_Position = u_projection * u_view * worldPosition;

  // 프래그먼트 셰이더로 텍스처 좌표를 전달합니다.
  v_texcoord = a_texcoord;

  v_projectedTexcoord = u_textureMatrix * worldPosition;

+  // 법선을 조정하여 프래그먼트 셰이더로 전달합니다.
+  v_normal = mat3(u_world) * a_normal;
+
+  // 표면의 월드공간 위치를 계산합니다.
+  vec3 surfaceWorldPosition = (u_world * a_position).xyz;
+
+  // 표면에서 조명을 향하는 벡터를 계산하고 프래그먼트 셰이더로 전달합니다.
+  v_surfaceToLight = u_lightWorldPosition - surfaceWorldPosition;
+
+  // 표면에서 뷰/카메라를 향하는 벡터를 계산하고 프래그먼트 셰이더로 전달합니다.
+  v_surfaceToView = u_viewWorldPosition - surfaceWorldPosition;
}
```

프래그먼트 셰이더:

```glsl
precision mediump float;

// 정점 셰이더에서 전달된 값
varying vec2 v_texcoord;
varying vec4 v_projectedTexcoord;
+varying vec3 v_normal;
+varying vec3 v_surfaceToLight;
+varying vec3 v_surfaceToView;

uniform vec4 u_colorMult;
uniform sampler2D u_texture;
uniform sampler2D u_projectedTexture;
uniform float u_bias;
+uniform float u_shininess;
+uniform vec3 u_lightDirection;
+uniform float u_innerLimit;          // 내적 공간에서의 값
+uniform float u_outerLimit;          // 내적 공간에서의 값

void main() {
+  // v_normal은 varying이기 때문에 보간되고, 단위 벡터가 아닐 수 있습니다.
+  // 정규화를 해서 다시 단위 벡터로 만들어줍니다.
+  vec3 normal = normalize(v_normal);
+
+  vec3 surfaceToLightDirection = normalize(v_surfaceToLight);
+  vec3 surfaceToViewDirection = normalize(v_surfaceToView);
+  vec3 halfVector = normalize(surfaceToLightDirection + surfaceToViewDirection);
+
+  float dotFromDirection = dot(surfaceToLightDirection,
+                               -u_lightDirection);
+  float limitRange = u_innerLimit - u_outerLimit;
+  float inLight = clamp((dotFromDirection - u_outerLimit) / limitRange, 0.0, 1.0);
+  float light = inLight * dot(normal, surfaceToLightDirection);
+  float specular = inLight * pow(dot(normal, halfVector), u_shininess);

  vec3 projectedTexcoord = v_projectedTexcoord.xyz / v_projectedTexcoord.w;
  float currentDepth = projectedTexcoord.z + u_bias;

  bool inRange =
      projectedTexcoord.x >= 0.0 &&
      projectedTexcoord.x <= 1.0 &&
      projectedTexcoord.y >= 0.0 &&
      projectedTexcoord.y <= 1.0;

  // 'r'채널에 깊이값이 저장되어 있습니다.
  float projectedDepth = texture2D(u_projectedTexture, projectedTexcoord.xy).r;
  float shadowLight = (inRange && projectedDepth <= currentDepth) ? 0.0 : 1.0;

  vec4 texColor = texture2D(u_texture, v_texcoord) * u_colorMult;
-  gl_FragColor = vec4(texColor.rgb * shadowLight, texColor.a);
+  gl_FragColor = vec4(
+      texColor.rgb * light * shadowLight +
+      specular * shadowLight,
+      texColor.a);
}
```

`shadowLight`를 `light`와 `specular` 효과의 양을 조절하기 위해 사용한 것에 주목하세요.
물체가 그림자 영역에 있다면 빛이 들지 않는것입니다.

이제 uniform들을 설정해 주기만 하면 됩니다.

```js
-function drawScene(projectionMatrix, cameraMatrix, textureMatrix, programInfo) {
+function drawScene(
+    projectionMatrix,
+    cameraMatrix,
+    textureMatrix,
+    lightWorldMatrix,
+    programInfo) {
  // 카메라 행렬로부터 뷰 행렬을 만듭니다.
  const viewMatrix = m4.inverse(cameraMatrix);

  gl.useProgram(programInfo.program);

  // 구와 평면에 모두 사용되는 유니폼을 설정합니다.
  // 주의: 셰이더에 대응되는 유니폼이 없는경우 무시됩니다.
  webglUtils.setUniforms(programInfo, {
    u_view: viewMatrix,
    u_projection: projectionMatrix,
    u_bias: settings.bias,
    u_textureMatrix: textureMatrix,
    u_projectedTexture: depthTexture,
+    u_shininess: 150,
+    u_innerLimit: Math.cos(degToRad(settings.fieldOfView / 2 - 10)),
+    u_outerLimit: Math.cos(degToRad(settings.fieldOfView / 2)),
+    u_lightDirection: lightWorldMatrix.slice(8, 11).map(v => -v),
+    u_lightWorldPosition: lightWorldMatrix.slice(12, 15),
+    u_viewWorldPosition: cameraMatrix.slice(12, 15),
  });

...

function render() {
  ...

-  drawScene(lightProjectionMatrix, lightWorldMatrix, m4.identity(), colorProgramInfo);
+  drawScene(
+      lightProjectionMatrix,
+      lightWorldMatrix,
+      m4.identity(),
+      lightWorldMatrix,
+      colorProgramInfo);

  ...

-  drawScene(projectionMatrix, cameraMatrix, textureMatrix, textureProgramInfo);
+  drawScene(
+      projectionMatrix,
+      cameraMatrix,
+      textureMatrix,
+      lightWorldMatrix,
+      textureProgramInfo);

  ...
}
```

설정된 유니폼 값들을 되짚어 봅시다.
[스포트라이트에 관한 글](webgl-3d-lighting-spot.html)을 떠올려보면 `innerLimit`와 `outerLimit`은 도트 공간(코사인 공간)의 값이고 조명의 방향을 따라서 뻗어나가는 형식이기 때문에 시야각의 절반만 필요합니다.
[카메라에 관한 글](webgl-3d-camera.html)에서 4x4 행렬의 세 번째 행이 Z축인 것을 기억하시면 `lightWorldMatrix`로부터 세 번째 행의 앞 세개 값을 가져오면 그것이 조명의 -Z방향이라는 것을 알 수 있습니다.
우리는 양의 방향이 필요하기 때문에 뒤집었습니다.
같은 글을 통해 네 번째 행이 월드공간 위치라는 것을 알고 있으므로 관련된 행렬로부터 `lightWorldPosition`과 `viewWorldPosition`(카메라의 월드 공간 위치)을 유사하게 얻을 수 있습니다.
물론 이 값들은 변수를 더 추가하거나 세팅할 수 있는 값을 추가해서도 얻을 수 있습니다.

배경도 검은색으로 바꾸고 절두체를 표시하는 선은 흰색으로 합시다.

```js
function render() {

  ...

  // 이제 깊이 텍스처를 장면에 투영하고 캔버스에 그립니다.
  gl.bindFramebuffer(gl.FRAMEBUFFER, null);
  gl.viewport(0, 0, gl.canvas.width, gl.canvas.height);
+  gl.clearColor(0, 0, 0, 1);
  gl.clear(gl.COLOR_BUFFER_BIT | gl.DEPTH_BUFFER_BIT);

  ...

  // ------ 절두체를 그립니다. ------
  {

    ...

    // 방금 계산한 유니폼 값들을 설정합니다.
    webglUtils.setUniforms(colorProgramInfo, {
-      u_color: [0, 0, 0, 1],
+      u_color: [1, 1, 1, 1],
      u_view: viewMatrix,
      u_projection: projectionMatrix,
      u_world: mat,
    });
```

이제 스포트라이트와 그림자를 얻었습니다.

{{{example url="../webgl-shadows-w-spot-light.html" }}}

방향성 조명에 대해서는 [방향성 조명과 관련된 글](webgl-3d-lighting-directional.html)에서 셰이더 코드를 복사하고 원근 투영을 직교 투영으로 바꾸면 됩니다.

먼저 정점 셰이더는,

```glsl
attribute vec4 a_position;
attribute vec2 a_texcoord;
+attribute vec3 a_normal;

-uniform vec3 u_lightWorldPosition;
-uniform vec3 u_viewWorldPosition;

uniform mat4 u_projection;
uniform mat4 u_view;
uniform mat4 u_world;
uniform mat4 u_textureMatrix;

varying vec2 v_texcoord;
varying vec4 v_projectedTexcoord;
varying vec3 v_normal;

-varying vec3 v_surfaceToLight;
-varying vec3 v_surfaceToView;

void main() {
  // 위치를 행렬과 곱합니다.
  vec4 worldPosition = u_world * a_position;

  gl_Position = u_projection * u_view * worldPosition;

  // 텍스처 좌표를 프래그먼트 셰이더로 넘겨줍니다.
  v_texcoord = a_texcoord;

  v_projectedTexcoord = u_textureMatrix * worldPosition;

  // 법선의 방향을 조정하고 프래그먼트 셰이더로 념겨줍니다.
  v_normal = mat3(u_world) * a_normal;

-  // 표면의 월드공간 위치를 계산합니다.
-  vec3 surfaceWorldPosition = (u_world * a_position).xyz;
-
-  // 표면에서 조명을 향하는 벡터를 계산하고 프래그먼트 셰이더로 전달합니다.
-  v_surfaceToLight = u_lightWorldPosition - surfaceWorldPosition;
-
-  // 표면에서 뷰/카메라를 향하는 벡터를 계산하고 프래그먼트 셰이더로 전달합니다.
-  v_surfaceToView = u_viewWorldPosition - surfaceWorldPosition;
}
```

프래그먼트 셰이더에서는,

```glsl
precision mediump float;

// 정점 셰이더에서 넘어온 값
varying vec2 v_texcoord;
varying vec4 v_projectedTexcoord;
varying vec3 v_normal;
-varying vec3 v_surfaceToLight;
-varying vec3 v_surfaceToView;

uniform vec4 u_colorMult;
uniform sampler2D u_texture;
uniform sampler2D u_projectedTexture;
uniform float u_bias;
-uniform float u_shininess;
-uniform vec3 u_lightDirection;
-uniform float u_innerLimit;          // 도트 공간
-uniform float u_outerLimit;          // 도트 공간
+uniform vec3 u_reverseLightDirection;

void main() {
  // v_normal은 varying이기 때문에 보간되고, 단위 벡터가 아닐 수 있습니다.
  // 정규화를 해서 다시 단위 벡터로 만들어줍니다.
  vec3 normal = normalize(v_normal);

+  float light = dot(normal, u_reverseLightDirection);

-  vec3 surfaceToLightDirection = normalize(v_surfaceToLight);
-  vec3 surfaceToViewDirection = normalize(v_surfaceToView);
-  vec3 halfVector = normalize(surfaceToLightDirection + surfaceToViewDirection);
-
-  float dotFromDirection = dot(surfaceToLightDirection,
-                               -u_lightDirection);
-  float limitRange = u_innerLimit - u_outerLimit;
-  float inLight = clamp((dotFromDirection - u_outerLimit) / limitRange, 0.0, 1.0);
-  float light = inLight * dot(normal, surfaceToLightDirection);
-  float specular = inLight * pow(dot(normal, halfVector), u_shininess);

  vec3 projectedTexcoord = v_projectedTexcoord.xyz / v_projectedTexcoord.w;
  float currentDepth = projectedTexcoord.z + u_bias;

  bool inRange =
      projectedTexcoord.x >= 0.0 &&
      projectedTexcoord.x <= 1.0 &&
      projectedTexcoord.y >= 0.0 &&
      projectedTexcoord.y <= 1.0;

  // 'r'은 깊이 값을 저장하고 있습니다.
  float projectedDepth = texture2D(u_projectedTexture, projectedTexcoord.xy).r;
  float shadowLight = (inRange && projectedDepth <= currentDepth) ? 0.0 : 1.0;

  vec4 texColor = texture2D(u_texture, v_texcoord) * u_colorMult;
  gl_FragColor = vec4(
-      texColor.rgb * light * shadowLight +
-      specular * shadowLight,
+      texColor.rgb * light * shadowLight,
      texColor.a);
}
```

유니폼들은,

```js
  // 구와 평면에 모두 사용되는 유니폼을 설정합니다.
  // 주의: 셰이더에 대응되는 유니폼이 없는 경우 무시됩니다.
  webglUtils.setUniforms(programInfo, {
    u_view: viewMatrix,
    u_projection: projectionMatrix,
    u_bias: settings.bias,
    u_textureMatrix: textureMatrix,
    u_projectedTexture: depthTexture,
-    u_shininess: 150,
-    u_innerLimit: Math.cos(degToRad(settings.fieldOfView / 2 - 10)),
-    u_outerLimit: Math.cos(degToRad(settings.fieldOfView / 2)),
-    u_lightDirection: lightWorldMatrix.slice(8, 11).map(v => -v),
-    u_lightWorldPosition: lightWorldMatrix.slice(12, 15),
-    u_viewWorldPosition: cameraMatrix.slice(12, 15),
+    u_reverseLightDirection: lightWorldMatrix.slice(8, 11),
  });
```

장면을 넓게 보기 위해 카메라를 조정하였습니다.

{{{example url="../webgl-shadows-w-directional-light.html"}}}

코드를 보면 명확한데 원래 방향성 조명은 방향만 가지고 위치는 없지만, 우리의 섀도우 맵은 너무 커서 특정 위치를 골라 섀도우 맵을 적용할 부분에 대해서만 계산을 수행하도록 되어 있습니다.

글이 너무 길어지고 있는데 여전히 그림자와 관련해서는 다룰 내용들이 많이 있습니다.
나머지는 [다음 글](webgl-shadows-continued.html)에서 알아보도록 합시다.

<div class="webgl_bottombar">
<a id="attachment-combinations"></a>
<h3>왜 사용하지 않는 색상 텍스처를 생성해야 하나요?</h3>
<p>여기서 우리는 WebGL 스펙의 세부 사항에 대해 묻습니다.</p>
<p>WebGL은 OpenGL ES 2.0을 기반으로 하고 기본적으로 <a href="https://www.khronos.org/registry/webgl/specs/latest/1.0/">WebGL 스펙</a>에 나열된 예외를 제외하고는 OpenGL ES 2.0을 따릅니다.</p>
<p>
프레임 버퍼를 만들 때 attachment를 추가하는데요.
모든 종류의 attachment를 추가할 수 있습니다.
위에선 RGBA/UNSIGNED_BYTE 텍스처 색상 attachment와 깊이 텍스처 attachment를 추가했습니다.
텍스처 렌더링에 관한 글에서 비슷한 색상 attachment를 첨부했지만 깊이 텍스처가 아닌 깊이 렌더 버퍼를 첨부했습니다.
또한 RGB 텍스처, LUMINANCE 텍스처, 그리고 기타 여러 유형의 텍스처와 렌더 버퍼를 첨부할 수도 있습니다.
</p>
<p>
<a href="">OpenGL ES 2.0 스펙</a>은 attachment의 특정 조합이 함께 작동하는지에 대한 많은 규칙을 제공합니다.
한 가지 규칙은 적어도 하나의 attachment는 있어야 한다는 겁니다.
또 다른 규칙은 모두 동일한 크기여야 합니다.
마지막은 규칙은,
</p>
<blockquote>
<h4>4.4.5 프레임 버퍼 완전성</h4>
<p>첨부된 이미지의 내부 포맷 조합은 <b>구현 종속</b> 제한을 위반하지 않습니다.</p>
</blockquote>
<p>이는 <b>attachment의 조합이 작동에 필요하지 않다</b>는 의미입니다.</p>
<p>
WebGL 위원회는 이를 보고 WebGL 구현이 최소 3개의 공통 조합을 지원하도록 요구하기로 결정했습니다.
<a href="https://www.khronos.org/registry/webgl/specs/latest/1.0/#6.8">WebGL 스펙의 섹션 6.8</a>:
<blockquote>
<ul>
  <li><code>COLOR_ATTACHMENT0</code> = <code>RGBA</code>/<code>UNSIGNED_BYTE</code> texture</li>
  <li><code>COLOR_ATTACHMENT0</code> = <code>RGBA</code>/<code>UNSIGNED_BYTE</code> texture + <code>DEPTH_ATTACHMENT</code> = <code>DEPTH_COMPONENT16</code> renderbuffer</li>
  <li><code>COLOR_ATTACHMENT0</code> = <code>RGBA</code>/<code>UNSIGNED_BYTE</code> texture + <code>DEPTH_STENCIL_ATTACHMENT</code> = <code>DEPTH_STENCIL</code> renderbuffer</li>
</blockquote>
<p>
나중에 <a href="https://www.khronos.org/registry/webgl/extensions/WEBGL_depth_texture/">WEBGL_depth_texture</a> 확장이 만들어졌습니다.
깊이 텍스처를 만들고 그걸 프레임 버퍼에 첨부할 수 있다는 것을 언급하지만 필요한 조합에 대해서는 더이상 언급하지 않는데요.
구현에 따라 어떤 조합이 작동할 수 있는지 알려주는 OpenGL ES 2.0 스펙 규칙이 주어지고, WebGL 스펙은 작동에 필요한 3가지 조합만을 나열합니다.
이러한 조합 중 어떤 것도 깊이 텍스처를 포함하지 않고 깊이 렌더 버퍼만 포함한다는 것은 깊이 텍스처를 사용을 보장하지 않는다는 겁니다.
</p>
<p>
실제로 대부분의 드라이버는 깊이 텍스처만 첨부하고 다른 attachment 없이 작동하는 것으로 보입니다.
하지만 2020년 2월 현재 사파리는 해당 조합이 작동하도록 허용하지 않습니다.
색상 attachment도 필요하며, 아마 <code>RGBA</code>/<code>UNSIGNED_BYTE</code> 색상 attachment가 필요할 겁니다.
그게 없으면 실패한다는 사실은 위 스펙에 포함되어 있습니다.
</p>
<p>
사파리에서 동작하려면 사용하지 않는 색상 텍스처가 필요하다는 것을 말하기 위한 기나긴 내용이었습니다.
또한 슬프게도 모든 드라이버/GPU/브라우저에서 작동하지 않는다는 것을 보장하지 않습니다.
다행히 해당 조합은 모든 곳에서 작동하는 것으로 보입니다.
또한 다행히도 <a href="https://webgl2fundamentals.org">WebGL2</a>의 기반이 되는 OpenGL ES 3.0은 스펙을 변경하여 작동하려면 더 많은 조합이 필요합니다.
하지만 2020년 2월 현재 <a href="https://webgl2fundamentals.org/webgl/lessons/webgl-getting-webgl2.html">사파리</a>는 WebGL2를 지원하지 않습니다.
따라서 WebGL1에서 사용하지 않는 색상 텍스처를 추가한 다음 기도해야 합니다. 😭
</p>
</div>

