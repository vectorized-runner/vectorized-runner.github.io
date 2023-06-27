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

# Implementation and the Challenges

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

We have 2 passes:

1. The *Deferred Geometry* pass, where we copy information from each object, at their fragment shader, to the *G-Buffers* (Textures).
2. The *Deferred Light* pass, this pass is global, it's not bound to one object. Actually in order to have this pass, we just render a full screen quad (so that we can run the Shader on the full screen-space).

So, the *Deferred Light* shader is the replacement of the *Forward Fragment* shader. 

Here's the main() of the Deferred Light:

```c++
void main()
{
    // retrieve data from gbuffer
    vec3 FragPos = texture(gPosition, TexCoords).rgb;
    vec3 Normal = texture(gNormal, TexCoords).rgb;
    vec3 Diffuse = texture(gAlbedoSpec, TexCoords).rgb;
    vec3 Specular = vec3(texture(gAlbedoSpec, TexCoords).a);
    
    vec3 totalDiffuse = vec3(0, 0, 0);
    vec3 totalSpecular = vec3(0, 0, 0);
    
    for(int i = 0; i < lightCount; ++i)
    {
        vec3 lightPos = lightPositions[i];
        float dsq = distancesq(lightPos, FragPos);
        vec3 I = lightIntensities[i] / dsq;
        vec3 L = normalize(lightPos - FragPos);
        vec3 V = normalize(cameraPos - FragPos);
        vec3 H = normalize(L + V);
        vec3 N = normalize(Normal);
        
        float NdotL = dot(N, L); // for diffuse component
        float NdotH = dot(N, H); // for specular component

        vec3 diffuseColor = I * Diffuse * max(0, NdotL);
        vec3 specularColor = I * Specular * pow(max(0, NdotH), 100);
        
        totalDiffuse += diffuseColor;
        totalSpecular += specularColor;
    }
    
    // vec3 ambientColor = Iamb * ka;
    vec3 ambientColor = vec3(0);

    FragColor = vec4(totalDiffuse + totalSpecular + ambientColor, 1);
}

```

Here's the main() of the Forward Fragment:

```c++
void main(void)
{
    vec3 totalDiffuse = vec3(0, 0, 0);
    vec3 totalSpecular = vec3(0, 0, 0);
    
    for(int i = 0; i < lightCount; i++){
        vec3 lightPos = lightPositions[i];
        vec3 pos = vec3(fragWorldPos);
        float dsq = distancesq(lightPos, pos);
        vec3 I = lightIntensities[i] / dsq;
        vec3 L = normalize(lightPos - pos);
        vec3 V = normalize(cameraPos - pos);
        vec3 H = normalize(L + V);
        vec3 N = normalize(fragWorldNor);

        float NdotL = dot(N, L); // for diffuse component
        float NdotH = dot(N, H); // for specular component

        vec3 diffuseColor = I * kd * max(0, NdotL);
        vec3 specularColor = I * ks * pow(max(0, NdotH), 100);
        
        totalDiffuse += diffuseColor;
        totalSpecular += specularColor;
    }
    
    // vec3 ambientColor = Iamb * ka;
    vec3 ambientColor = vec3(0);
    
    fragColor = vec4(totalDiffuse + totalSpecular + ambientColor, 1);
}

```

Looks pretty similar, isn't it? The only difference is, on the Deferred shader we sample from the G-buffers, whereas on the Forward shader we get the input from the Vertex shader.

# Visuals Comparison

There aren't a lot of objects in the Scene, only Armadillos (enemies) and Pillars (scaled cubes) so that I can see.

Forward:

<img src="{{site.url}}/images/forward.png" width = "800" height = "600" style="display: block; margin: auto;" />

Deferred:

<img src="{{site.url}}/images/deferred.png" width = "800" height = "600" style="display: block; margin: auto;" />

As you can see, except of a few artifacts, the render is nearly the same.

# Benchmarks

Forward rendering wins on low light counts, but Deferred rendering scales much better as I push towards 200+ lights.

Here are the some average results:

**Render Milliseconds: 42.699814 Mode: Forward LightCount: 30**

**Render Milliseconds: 41.610718 Mode: Deferred LightCount: 30**

--

**Render Milliseconds: 85.006714 Mode: Forward LightCount: 251**

**Render Milliseconds: 63.560486 Mode: Deferred LightCount: 251**

# The Missing Implementation

Admittedly, I haven't spent enough time on this hw, so some of the implementation is missing, you'll have to imagine it :)

1. The player doesn't have a gun model
2. The enemies don't actually die when hit
3. There's no ground, but there used to be one, but the deferred rendering implementation broke everything

# A Final Note

I've finished the HW while on vacation and barely had any internet connection, had to find a cafe with a wi-fi so that I could submit the homework at the last second :)

Even though this semester was really though on me, I've learned a lot in this course. Leaving the last project to us was really a treat. Thanks to Ahmet and Kadir hoca for everything.

Hope to see you in another course. I think we all deserved to take a break.



<img src="{{site.url}}/images/final-note.jpeg" width = "800" height = "600" style="display: block; margin: auto;" />



