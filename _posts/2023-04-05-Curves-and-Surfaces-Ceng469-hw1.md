This homework has been a quite ride for me. It took me so long to actually render a triangle on the surface I was almost quite giving up :) But once I've seen something on the screen with the correct transformations rendered I've got enough courage to pull it off.

# Challenges and Solutions

## Running on Mac

Initially I couldn't get OpenGL version 460 running on my Mac. I thought it was mandatory to run on 460, but then I learned that's not the case, but the major version should be 4. I got my window hints right on Mac with this code:

```c++
void AddWindowHints(){
    // TODO: OpenGL 460 doesn't work on Mac, uncomment later.
    // glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR, 4);
    // glfwWindowHint(GLFW_CONTEXT_VERSION_MINOR, 6);
    // glfwWindowHint(GLFW_OPENGL_PROFILE, GLFW_OPENGL_CORE_PROFILE);
    glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR, 4);
    glfwWindowHint(GLFW_CONTEXT_VERSION_MINOR, 1);
    glfwWindowHint(GLFW_OPENGL_PROFILE, GLFW_OPENGL_CORE_PROFILE);
    glfwWindowHint(GLFW_OPENGL_FORWARD_COMPAT, GL_TRUE);
}
```

## Drawing the First Triangle

I wanted to remember the whole OpenGL pipeline, so I've started writing the code from scratch, instead of copy-pasting code from 477 homeworks, as I knew I'd hit a barrier and not knowing the context would block me later. It took me a lot of commits but I've managed to render my first rectangle with shared indices.

<img src="{{site.url}}/images/rect.png" width = "400" height = "400" style="display: block; margin: auto;" />

## Generating the Initial Grid

Before doing any curve stuff, I needed to handle matching vertices and indices for a simple grid. At first, I've tried to make the most optimized setup that wastes no indices, but that turned out to be harder than I thought, so I fell back to simpler algorithm.

```c++
void UpdateMesh(){
    
    // TODO: Handle parsing these from the file
    const float minX = -0.5f;
    const float maxX = 0.5f;
    const float minZ = -0.5f;
    const float maxZ = 0.5f;
    
    vertices.reserve(maxVertexCount);
    vertices.clear();
    
    indices.reserve(maxIndexCount);
    indices.clear();
    
    for(int ix = 0; ix < sampleCount; ix++){
        for(int iz = 0; iz < sampleCount; iz++){
            auto index = sampleCount * iz + ix;
            auto tX = ix / (float)(sampleCount - 1);
            auto posX = lerp(minX, maxX, tX);
           
            // TODO: Handle position after parsing the file
            auto posY = 0.0f;

            auto tZ = iz / (float)(sampleCount - 1);
            auto posZ = lerp(minZ, maxZ, tZ);
            
            auto point = vec3(posX, posY, posZ);
            
            // TODO: Handle normal after parsing the file
            auto normal = vec3(0.0f, 1.0f, 0.0f);
            
            vertices[index] = VertexData(point, normal);
        }
    }
     
    for(int ix = 0; ix < sampleCount - 1; ix++){
        for(int iz = 0; iz < sampleCount - 1; iz++){
            int index = sampleCount * iz + ix;
            indices.push_back(index);
            indices.push_back(index + 1);
            indices.push_back(index + sampleCount);
            indices.push_back(index + 1);
            indices.push_back(index + sampleCount + 1);
            indices.push_back(index + sampleCount);
        }
    }
}
```

<img src="{{site.url}}/images/initial-grid.png" width = "400" height = "400" style="display: block; margin: auto;" />

## Generating the input1.txt

I wanted to just hardcode everything and get the curve on input1.txt working, without dealing with file parsing, so I've hardcoded the values, and spent time on how to design the data for shaders instead. Also I'm skipping normals at this point.

```c++
uniform float[16] curveHeights;
uniform float s;
uniform float t;
...
  
float sampleHeightOnCurve(float s, float t){
    float result = 0.0f;
    
    for (int i = 0; i <= 3; i++)
    for (int j = 0; j <= 3; j++)
    {
        float height = curveHeights[i * 3 + j];
        result += height * bernstein(3, i, s) * bernstein(3, j, t);
    }

    return result;
}

void main()
{
    float height = sampleHeightOnCurve(s, t);
    vec3 finalVertex = vec3(inVertex.x, height, inVertex.z) * coordsMultiplier;
    gl_Position = projection * view * model * vec4(finalVertex, 1);
    
    // TODO: Remove this
    ourColor = vec3(0, 1, 0);

    // TODO:
    normal = sampleNormalOnCurve(s, t);
}
```

and here's the output:

<img src="{{site.url}}/images/initial-curve.png" width = "400" height = "400" style="display: block; margin: auto;" />

## Designing for the actual Input Files

So I've needed to setup my data for potentially 36 curves. My initial thought was to just allocate whole memory up front. It turned out to be a good decision. I've done this on the CPU side as well.

Also I needed to pass the CurveIndex to the Vertex shader so that the vertex would know which index to sample from.

```c++
uniform float curveHeights[16 * 36];

layout (location = 0) in vec3 inVertex;   // the position variable has attribute position 0
layout (location = 1) in vec3 stc; // the s, t, curve index

float sampleHeightOnCurve(vec3 stc){
    float s = stc.x;
    float t = stc.y;
    int curveIndex = int(stc.z);
    float result = 0.0f;
    
    for (int cx = 0; cx < 4; cx++)
    for (int cz = 0; cz < 4; cz++)
    {
        float height = curveHeights[curveIndex * 16 + cx * 4 + cz];
        result += height * bernstein(3, cx, s) * bernstein(3, cz, t);
    }

    return result;
}
```

I also needed to locally-calculate s, t of the surfaces, as per surface it needs to go in the [0, 1] range:

```c++
    for(int ix = 0; ix < sampleCount; ix++){
        for(int iz = 0; iz < sampleCount; iz++){
            auto curveIndexX = ix / (sampleCount / curveCountX);
            auto curveIndexZ = iz / (sampleCount / curveCountZ);
            auto curveIndex = curveIndexZ * curveCountX + curveIndexX;
            
            cout << "CurveIndex for  " << ix << ", " << iz << " is " << curveIndex << endl;
            
            auto index = sampleCount * iz + ix;
            auto tX = ix / (float)(sampleCount - 1);
            auto posX = lerp(minX, maxX, tX);

            // This is overridden
            auto posY = 0.0f;

            auto tZ = iz / (float)(sampleCount - 1);
            auto posZ = lerp(minZ, maxZ, tZ);
            
            auto point = vec3(posX, posY, posZ);
            
            auto tileZ = (float)(sampleCount - 1) / curveCountZ;
            auto s = fmod(iz, tileZ) / tileZ;
            
            auto tileX = (float)(sampleCount - 1) / curveCountX;
            auto t = fmod(ix, tileX) / tileX;
                                                     
            DebugAssert(s >= 0.0f && s <= 1.0f, "SRange");
            DebugAssert(t >= 0.0f && t <= 1.0f, "TRange");
            
            auto stc = vec3(s, t, curveIndex);
            
            vertices[index] = VertexData(point, stc);
        }
    }
```

## Hitting the OpenGL uniform Limit

I've set up my data in **Structure of Arrays** fashion:

```c++
uniform float curveHeights[16 * 36];
uniform float curvePositionX[16 * 36];
uniform float curvePositionZ[16 * 36];
```

Then my shader wouldn't compile, it said the code has hit the uniform limit. Just merging into one struct worked out in the end:

```c++
uniform vec3 curvePositions[16 * 36];
```

## A small error in the Surface Normal, losing my mind for a day

I've had an error with the partial derivatives where I've used ```s * s``` instead of ```-s * s```, which took a lot of hair pulling to find out. 

The surface looked like this:

<img src="{{site.url}}/images/broken-normals.png" width = "400" height = "400" style="display: block; margin: auto;" />

## Honorable Mentions: Getting it working on Ineks

1. GLFW_KEY_SPACE wouldn't work on Inek (I used this for wireframe toggle).
2. As I decreased the SampleCount, my curve would get smaller on Linux, while it was fine on Mac. This turned out to be an issue with ```pow``` function. I've implemented the integer version of ```pow``` myself. 
3. ```glm::lerp``` was giving compile errors for some reason. I've embedded my own code for this as well.

# Conclusion

I've learned a lot in this homework, however, some parts were really frustrating, such as input files potentially including multiple bezier curves, that took a lot of time to handle and I think wasn't contributing much to actually learning about the surfaces.

Here's a final render:

<img src="{{site.url}}/images/final-render.png" width = "400" height = "400" style="display: block; margin: auto;" />