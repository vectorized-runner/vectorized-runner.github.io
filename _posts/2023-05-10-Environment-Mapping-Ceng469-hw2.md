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

I've added a method to easily draw any mesh:

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



TODO:

TODO: 

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
    
    if(input.move != 0){
        auto carAccel = Car::InputAcceleration * input.move;
        auto deltaSpeed = carAccel * gameTime.deltaTime;
        car.speed += deltaSpeed;
    }
    else{
        auto direction = car.speed > 0.0f ? -1 : 1;
        auto carAccel = Car::Deceleration * direction;
        auto deltaSpeed = carAccel * gameTime.deltaTime;
        if(abs(deltaSpeed) > abs(car.speed)){
             deltaSpeed = -car.speed;
        }
    }
    
    auto carVelocity = tf.Forward() * car.speed;
    tf.position += carVelocity * gameTime.deltaTime;
}
```





