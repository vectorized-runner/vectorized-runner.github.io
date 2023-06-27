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



