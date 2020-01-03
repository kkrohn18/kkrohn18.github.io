---
layout: post
title: WebGL Refraction Demo
#image: /img/hello_world.jpeg
gh-repo: kkrohn18/RefractionDemo
---

# Typescript + WebGL
### A live demo can be found [here](https://kkrohn18.github.io/RefractionDemo/index.html)

This is a demonstration of using WebGL, a graphics library based on OpenGL, and Typescript to simulate single-sided refraction of light passing through an object.

# How it works
### Render Loop


```typescript
function render() {

    
    gl.clearColor(1.0, 1.0, 1.0, 1.0);
    gl.clear(gl.COLOR_BUFFER_BIT | gl.DEPTH_BUFFER_BIT);
    gl.viewport(0, 0, gl.drawingBufferWidth, gl.drawingBufferHeight);

    let p:mat4 = perspective(45.0, 1, 1.0, 100.0);



    gl.useProgram(skyboxProgram);
    gl.uniformMatrix4fv(uproj[0], false, p.flatten());
    let mv:mat4 = lookAt(new vec4(0, 0, 4.5, 1), new vec4(0, 0, 0, 1), new vec4(0, 1, 0, 0));

    // camera position that will be sent to the envMapProgram
    let cameraPosition:vec4 = new vec4(0, 0, 4.5, 1);
    // rotation matrix that will be sent to the envMapProgram
    let rotation:mat4 = mv = mv.mult(rotateY(yAngle).mult(rotateX(xAngle)));

    mv = mv.mult(scalem(10, 10, 10));
    gl.uniformMatrix4fv(umv[0], false, mv.flatten());
    skybox.draw();



    gl.useProgram(envMapProgram);
    gl.uniformMatrix4fv(uproj[1], false, p.flatten());
    let objectmv:mat4 = lookAt(new vec4(0, 0, 7, 1), new vec4(0, 0, 0, 1), new vec4(0, 1, 0, 0));
    gl.uniformMatrix4fv(umv[1], false, objectmv.flatten());

    gl.uniformMatrix4fv(uRotation, false, rotation.flatten());
    gl.uniform4fv(uCameraPosition, cameraPosition);
    sphere.draw();

}
```
The render function gets called with each frame update.
It clears the canvas and redraws the updated matrices on the canvas

First the skybox environment is rendered using the vertex shader and fragment shader found in SkyboxShaders. 
- The skybox consists of 6 images each rendered to a texture and then applied to each side of the cube.
- The images of the skybox are from: [OpenGameArt.org](https://opengameart.org/content/elyvisions-skyboxes)

```glsl
// Vertex shader
in vec4 vPosition;

out vec4 ftexCoord;

uniform mat4 model_view;
uniform mat4 projection;

void main() {

    ftexCoord = vPosition;

    gl_Position = projection * model_view * vPosition;

}
```
Then the data is passed to the fragment shader to calculate where each value will be in terms of pixels on the canvas
```glsl
// Fragment shader
precision mediump float;

in vec4 ftexCoord;

uniform samplerCube textureSampler;

out vec4 fColor;

void main() {

    fColor = texture(textureSampler, ftexCoord.xyz);
}
```

After the skybox is drawn on the screen, the render function switches to the shader program used to render the sphere with refraction effects (found in RefractionShaders)
```glsl
// Vertex shader
in vec4 vPosition;
in vec4 vNormal;

uniform mat4 projection;
uniform mat4 model_view;

uniform mat4 rotation;
uniform vec4 cameraPosition;

out vec3 V;
out vec3 N;

void main() {

    vec4 veyepos = rotation * vPosition;

    V = normalize(veyepos.xyz - cameraPosition.xyz);
    N = normalize(rotation*vNormal).xyz;

    gl_Position = projection * model_view * vPosition;
}
```
Next, the data is passed through the Fragment shader
```glsl
// Fragment shader
precision mediump float;

in vec4 veyepos;
in vec3 V;
in vec3 N;

uniform samplerCube objectTexture;

uniform float indexOfRefraction;

out vec4 fColor;

void main() {

    vec3 fV = normalize(V);
    vec3 fN = normalize(N);

    // 1.00 for air
    float ratio = 1.00 / indexOfRefraction;
    vec3 refracted = refract(fV, fN, ratio);

    fColor = texture(objectTexture, refracted);
}
```
The texture of the skybox environment is mapped onto the sphere and the texture coordinates used to map it are calculated here in the fragment shader. The built-in GLSL refract function takes in a normal vector on the sphere, the incident eye vector to the surface of the sphere, and the index of refraction of the surface material. The normal vectors and incident eye vectors were calculated in "world space" and passed in from the vertex shader. The resulting vector that returns from the refract function is then used as the texture coordinates for the surface of the object to simulate single-sided refraction.
