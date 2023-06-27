We're *finally* here! It's the last and the most exciting homework of the semester, as I choose it myself: **Deferred Rendering**

First I want to recap why Deferred Rendering is important: 

- In realtime simulations, doing realistic light calculations is really costly (on the fragment shader as well!).
- In games, we want to run at least 60 frames per second nowadays.
- In a scene with thousands of objects, only the closest object is going to be drawn on the pixel, but we're running light calculation on all of the overlapping objects (if not culled completely). We're wasting a lot of processor time.

We can do better: *Defer* the heavy calculations to a later stage.

This technique is practical and used in commercial engines: The **Unreal Engine** uses Deferred Rendering by default.

# About the Project

Main goal about the project was to test the performance and visuals of Forward vs. Deferred rendering, in both low and high light count scenarios.

I've also wanted to implement it as a game: A simple first-person shooter, where you can move with a character. But there's a twist: The Player throws lights out of his gun.

# Challenges, Implementation, Bugs

## Creating the Benchmark Setup

Most important technical requirements for the project are:

1. Capture render time only, don't include simulation time, so that it won't affect the benchmarks
2. Ability to pause the game to I can render the exact scene with both forward and deferred rendering

So I needed to completely decouple simulation logic from the rendering logic. I've done this in the previous homeworks as well, so it's trivial to me. Here's the code for the main loop:

```c++
void ProgramLoop(GLFWwindow* window){
    while (!glfwWindowShouldClose(window))
    {
        UpdateInput(window);
        
        if(!simulationPaused){
            RunSimulation();
        }
        
        auto renderBegin = GetCurrentTime();
        Render(window);
        auto renderEnd = GetCurrentTime();
        auto renderDt = renderEnd - renderBegin;
        auto renderMs = renderDt * 1000;
        auto modeText = renderDeferred ? "Deferred" : "Forward";
        auto lightCount = to_string(scene.lightCount);
        cout << "Render Milliseconds: " << to_string(renderMs) << " Mode: " << modeText << " LightCount: " << lightCount << endl;
    }
}
```

as you can see, I can completely pause the simulation, and the benchmark time only includes rendering.

## Handling the Render Switch

For each **Mesh**, I now have *Deferred* and *Forward* shaders.

```c++
struct Mesh {
    string path;
    vector<Vertex> vertices;
    vector<Normal> normals;
    vector<Texture> textures;
    vector<Face> faces;
    GLuint gVertexAttribBuffer;
    GLuint gIndexBuffer;
    int gVertexDataSizeInBytes;
    int gNormalDataSizeInBytes;
    int vbo;
    int vao;
    Shader forwardShader;
    Shader deferredShader;
};
```

and to draw this mesh, I just use the global setting:

```c++
void DrawObject(const mat4& projectionMatrix, const mat4& viewingMatrix, const Object& obj, bool deferred, int lightIndex) {
    const auto modelingMatrix = obj.transform.GetMatrix();
    
    for(int i = 0; i < obj.meshIndices.size(); i++){
        auto& mesh = GetMesh(obj.meshIndices[i]);
        auto shader = deferred ? mesh.deferredShader : mesh.forwardShader;
        DrawMesh(projectionMatrix, viewingMatrix, modelingMatrix, mesh, shader, lightIndex);
    }
}
```

I think this worked out to be pretty nice.

## Having 2 Shaders for each Mesh

It was hard to completely get the exact visuals for both of the Shaders. 

First, let me give you a quick overview of how deferred shading works:

- 