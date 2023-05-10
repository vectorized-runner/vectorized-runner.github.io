Here we go again with another exciting homework. This one took longer than the first. I'm also not sure if I did some calculations correctly, but today's the deadline so I'm posting what I've accomplished.

# Challenges and Decisions

## Common Components

I've included common components used in games in order to abstract away commonly used logic.

### Mesh

A mesh basically abstracts away lowest-level constructs in order to draw a single model.

```c++
struct Mesh {
    vector<Vertex> vertices;
    vector<Normal> normals;
    vector<Texture> textures;
    vector<Face> faces;
    GLuint gVertexAttribBuffer;
    GLuint gIndexBuffer;
    int gVertexDataSizeInBytes;
    int gNormalDataSizeInBytes;
    int shaderId;
    int vbo;
    int vao;
    Shader shader;
};
```

I've added a method to easily draw any mesh (implementation not included):

```c++
void DrawMesh(const mat4& projectionMatrix, const mat4& viewingMatrix, const mat4& modelingMatrix, const Mesh& mesh);
```

### Transform

Transform is very commonly used in games. Instead of working on matrices, it's more intuitive to manipulate position, rotation, scale separately. We can compute the matrix at any time.

```c++
struct Transform{
    vec3 position = vec3(0, 0, 0);
    quat rotation = quat(1, 0, 0, 0);
    vec3 scale = vec3(1, 1, 1);
    
    mat4 GetMatrix() const {
        auto id = glm::mat4(1.0f);
        auto t = glm::translate(id, position);
        auto r = glm::toMat4(rotation);
        auto s = glm::scale(id, scale);
        
        return t * r * s;
    }
    
    vec3 Up() const {
        return rotate(rotation, vec3(0, 1, 0));
    }
    
    vec3 Forward() const {
        return rotate(rotation, vec3(0, 0, 1));
    }
    
    vec3 Right() const {
        return rotate(rotation, vec3(1, 0, 0));
    }
};
```

### Object

The object is used to make an hierarchy of multiple meshes. For example, we can think of the ***Car*** as an object with of body, window and tire meshes.

```c++
struct Object {
    Transform transform;
    vector<Mesh> meshes;
};
```

Then, we can embed this object into our commonly-used shapes:

```c++
struct Ground {
    Object obj;
};

struct Statue{
    Object obj;
};

struct Car {
    float speed = 0.0f;
    
    Object obj;
    
    static constexpr float InputAcceleration = 25.0f;
    static constexpr float Deceleration = 10.0f;
};
```

We can draw an object easily now:

```c++
void DrawObject(const mat4& projectionMatrix, const mat4& viewingMatrix, const Object& obj) {
    const auto modelingMatrix = obj.transform.GetMatrix();
    
    for(int i = 0; i < obj.meshes.size(); i++){
        DrawMesh(projectionMatrix, viewingMatrix, modelingMatrix, obj.meshes[i]);
    }
}
```

### Camera

Creating a custom component for camera helped me to wrap my head around calculations, especially when rendering cubemaps:

```c++
struct Screen{
    int width = 800;
    int height = 600;
};

struct Camera{
    
    vec3 position;
    vec3 lookPosition;
    vec3 up = vec3(0, 1, 0);
    
    float fovYDegrees = 45.0f;
    float near = 0.0001f;
    float far = 10000.0f;
    Screen screen;
    
    mat4 GetViewingMatrix(){
        return lookAt(position, lookPosition, up);
    }
    
    mat4 GetProjectionMatrix(){
        float fovYRadians = radians(fovYDegrees);
        float aspect = screen.width / (float)screen.height;
        return perspective(fovYRadians, aspect, near, far);
    }
};
```



## Update Loop

I've made the loop similar to Game Engines, the simulation data updates first, then render loop renders each object using their simulation data.

```c++
void ProgramLoop(GLFWwindow* window){
    while (!glfwWindowShouldClose(window))
    {
        UpdateLogic();
        Render(window);
    }
}
```

Updating the Logic is divided into 3 parts,

```c++
void UpdateLogic(){
    UpdateTime();
    UpdateCar();
    UpdateCamera();
}
```

We need to update the time first in order to get the **deltaTime**, so that our simulation runs frame-rate independent fashion.

```c++
void UpdateTime() {
    auto newTime = (float)glfwGetTime();
    gameTime.deltaTime = newTime - gameTime.time;
    gameTime.time = newTime;
}
```

Then, I'm updating the car depending on User Input, applying constant **acceleration** to the car.

I've also added **deceleration** to stop the car when the User isn't providing any input.

```c++
void UpdateCarPosition(){
    auto& tf = car.obj.transform;
    
    DebugAssert(IsLengthEqual(tf.Forward(), 1.0f), "CarVelocity");
    auto deltaSpeed = 0.0f;
    
    if(input.move != 0){
        auto carAccel = Car::InputAcceleration * input.move;
        deltaSpeed = carAccel * gameTime.deltaTime;
        car.speed += deltaSpeed;
    }
    else{
        auto direction = car.speed > 0.0f ? -1 : 1;
        auto carAccel = Car::Deceleration * direction;
        deltaSpeed = carAccel * gameTime.deltaTime;
        if(abs(deltaSpeed) > abs(car.speed)){
             deltaSpeed = -car.speed;
        }
        
    }
  
    car.speed += deltaSpeed;
    auto carVelocity = tf.Forward() * car.speed;
    tf.position += carVelocity * gameTime.deltaTime;
}
```

The Camera always looks at the Car, and depending on the input **direction** we determine its position, offset up by a little bit for better viewing experience.

```c++
void UpdateCamera(){
    const float distance = 20.0f;
    const float upOffset = 5.0f;
    
    vec3 targetPos;
    auto carTf = car.obj.transform;
    
    switch(input.direction){
        case Direction::Left:{
            targetPos = carTf.position + carTf.Right() * distance;
            break;
        }
        case Direction::Back:{
            targetPos = carTf.position - carTf.Forward() * distance;
            break;
        }
        case Direction::Front:{
            targetPos = carTf.position + carTf.Forward() * distance;
            break;
        }
        case Direction::Right:{
            targetPos = carTf.position - carTf.Right() * distance;
            break;
        }
    }
    
    targetPos += vec3(0, 1, 0) * upOffset;
    
    camera.position = targetPos;
    camera.lookPosition = carTf.position;
}
```



## Input

I couldn't handle input properly first, as the program loop runs at a higher frequency than the **glfwSetKeyCallback**.

Then I realized it also reports **key up** actions as well. So I can handle it precisely if I only track press and releases, discarding the repeats.

```c++
		// ...   

    if(action == GLFW_REPEAT){
        return;
    }
    
    auto isPress = action == GLFW_PRESS;
		if(key == GLFW_KEY_W){
        if(isPress){
            input.move = 1;
        }
        else{
            input.move = 0;
        }
    }
    else if(key == GLFW_KEY_S){
        if(isPress){
            input.move = -1;
        }
        else{
            input.move = 0;
        }
    }
```



## Rendering

Rendering meshes were trivial as their code is set up from the previous homeworks.

The Render loop is pretty straightforward:

```c++
void Render(GLFWwindow* window){
    ClearScreen();
    
    DrawDynamic();
    
    DrawSkybox();
    
    auto projectionMatrix = camera.GetProjectionMatrix();
    auto viewingMatrix = camera.GetViewingMatrix();
    DrawObject(projectionMatrix, viewingMatrix, car.obj);
    DrawObject(projectionMatrix, viewingMatrix, armadillo.obj);
    DrawObject(projectionMatrix, viewingMatrix, bunny.obj);
    DrawObject(projectionMatrix, viewingMatrix, teapot.obj);
    
    DrawGround(projectionMatrix, viewingMatrix);
    
    glfwSwapBuffers(window);
    glfwPollEvents();
}
```

I'm pretty sure my dynamic cubemap calculations are incorrect, but I couldn't find what's causing it.

Also there are some weird artifacts on statues while moving the car.

<img src="{{site.url}}/images/hw2-img1.png" width = "800" height = "600" style="display: block; margin: auto;" />

<img src="{{site.url}}/images/hw2-img2.png" width = "800" height = "600" style="display: block; margin: auto;" />

<img src="{{site.url}}/images/hw2-img3.png" width = "800" height = "600" style="display: block; margin: auto;" />



## Conclusion

This homework taught me a lot, especially using Depth and Frame buffers and putting it into practice was nice.

If I had more time I'd like to play around with different models and maybe make this a complete game, maybe later :)