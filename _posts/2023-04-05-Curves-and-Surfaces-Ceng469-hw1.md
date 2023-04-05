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

<img src="{{site.url}}/images/initial_grid.png" width = "400" height = "400" style="display: block; margin: auto;" />