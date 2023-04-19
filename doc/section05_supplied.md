Supplied Services
=================

Indeed, when a service is single, `service()` won't accept arguments to forward to the constructor.
This is because only the first call to `service()` creates an instance. Consider this code (bad example):

```c++
// Bad :(
Scene& scene1 = c.service<SceneService>(1024, 768);
Scene& scene2 = c.service<SceneService>(1920, 1080); // Oops! Constructor not called! The instance is reused.

// The scene still have a resolution of 1024x768
```

As a matter of fact, the `service` function has no means of telling if the service has actually been created here.
To resolve this, you can instead request the constuction of a service with the `emplace(...)` function.
That function will perform injection like `service()`, but serves only to save a single into the container.
Just like `std::map`, emplace only perform initialization if the container don't contain and instance yet.
It returns `true` if the service is created, and `false` if there's already an instance in the container:

```c++
bool inserted = c.emplace<SceneService>(1920, 1080); // (1)
assert(inserted); // Passes.

inserted = c.emplace<SceneService>(1024, 768);
assert(inserted); // Fires, not created two times.

Scene& scene = container.service<SceneService>(); // Returns the instance created at (1)
```

As we can see, `emplace` only constructs if the element is not found.
But as opposed to `service`, it can report if it has already been inserted before.
This gives us the opportunity to handle those case correctly.

## Supplied Services

As `container.service<SceneService>()` requires the `SceneService` to be constructible using only it's dependencies, the following `Scene` declaration won't work:

```c++
struct Scene {
    Scene(Camera c, int w, int h) :
        camera{c}, width{w}, height{h} {}
    
private:
    Camera camera;
    int width;
    int height;
};

Scene& scene = container.service<SceneService>(); // fails!
```

Its constructor must receive two additional integers after its dependencies.
Since `container.service<SceneService>()` must create a scene and does not forward any arguments to single services,
there is no way to obtain a scene from the container!

In these cases, we must tell the container that it's normal that it cannot construct the service without arguments,
and must be provided to the container before usage.

```c++
struct SceneService : kgr::single_service<Scene, kgr::dependency<CameraService>>, kgr::supplied {};
```

Now we can use the service like this:

```c++
container.emplace<SceneService>(1920, 1080); // contruct a scene in the container.

Scene& scene = container.service<SceneService>(); // works, won't try to construct it.
```

Note that if the instance is not found, the container won't be able to construct it and will throw a `kgr::supplied_not_found` instead.

## External services

The kangaru library also provides two supplied services specialized for cases where instances are created from an external system.

The first, `kgr::extern_service<T>` holds a reference to an instance of `T`. It behaves like a single service but the instance must be provided to the container manually.

Here's an example of extern service:

```c++
struct Scene {};

struct SceneService : kgr::extern_service<Scene> {};

int main() {
    Scene scene;
    kgr::container container;
    
    // Add the scene
    container.emplace<SceneService>(scene);
    
    // Passes, the container returns the instance we sent it.
    assert(&scene == &container.service<SceneService>());
}
```

The other external service is `kgr::extern_shared_service<T>`, which is analogous to the `kgr::extern_service` but is injected and contains the service by shared pointers.

Here's the same example as above, but with the shared external service:

```c++
struct Scene {};

struct SceneService : kgr::extern_shared_service<Scene> {};

int main() {
    auto scene = std::make_shared<Scene>();
    kgr::container container;
    
    // Add the scene
    container.emplace<SceneService>(scene);
    
    // Passes, the container returns the same shared pointer we sent it.
    assert(scene == container.service<SceneService>());
}
```

## Replace Services

The `emplace` function will only construct a service is it's not in the container yet. But what if you wanted to replace an existing service?

The container gives you a way to explicitly replace a single service to be used in future injection. Its usage is similar to emplace:

```c++
kgr::container container;

container.emplace<SceneService>(640, 480);
Scene& scene1 = container.service<SceneService>();
container.replace<SceneService>(1920, 1080);
Scene& scene2 = container.service<SceneService>();

assert(&scene1 != &scene2); // passes, these are different instances of scenes
```

Note that when replacing a service the container will not destroy the old service. In our example, `scene1` is still valid after `replace` has been called.
The lifetime of the first `SceneService` is not affected and will be destroyed at the same time as the container.

To see more about supplied services, please see [example5](../examples/example5/example5.cpp).

[Next chapter: Autocall](section06_autowire.md)
