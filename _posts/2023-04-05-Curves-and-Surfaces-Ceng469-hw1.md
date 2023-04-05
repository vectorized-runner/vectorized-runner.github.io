This homework has been a quite ride for me. It took me so long to actually render a triangle on the surface I was almost quite giving up :) But once I've seen something on the screen with the correct transformations rendered I've got enough courage to pull it off.

# Challenges and Solutions

## Running on Mac

Initially I couldn't get OpenGL version 460 running on my Mac. I thought it was mandatory to run on 460, but then I learned that's not the case, but the major version should be 4. I got my window hints right on Mac with this:

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

// TODO: Make the image work here

<img src="{{site.url}}/images/rect.png" style="display: block; margin: auto;" />

xx







