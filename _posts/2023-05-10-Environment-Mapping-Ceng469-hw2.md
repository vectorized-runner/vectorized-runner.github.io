Here we go again with another exciting homework. This one took longer than the first. I'm also not sure if I did some calculations correctly, but today's the deadline so I'm posting what I've accomplished.

# Challenges and Decisions

# Update Loop

I've made the loop similar to Game Engines, the simulation data updates first (input handling, moving the car), then render loop renders each object using their simulation data.

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





