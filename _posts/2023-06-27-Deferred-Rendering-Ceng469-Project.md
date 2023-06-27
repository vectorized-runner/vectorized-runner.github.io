We're *finally* here! It's the last and the most exciting homework of the semester, as I choose it myself: **Deferred Rendering**

First I want to recap why Deferred Rendering is important: 

- In realtime simulations, doing realistic light calculations is really costly.
- In games, we want to run at least 60 frames per second.
- In a scene with thousands of objects, only the closest object is going to be drawn on the pixel, but we're running light calculation on all of overlapping objects. We're wasting a lot of processor time.

We can do better: *Defer* the heavy calculations to a later stage.

# Implementation

