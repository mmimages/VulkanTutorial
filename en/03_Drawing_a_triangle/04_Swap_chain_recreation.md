## Introduction

The application we have now successfully draws a triangle, but there are some
circumstances that it isn't handling properly yet. It is possible for the window
surface to change such that the swap chain is no longer compatible with it. One
of the reasons that could cause this to happen is the size of the window
changing. We have to catch these events and recreate the swap chain.

## Recreating the swap chain

Create a new `recreateSwapChain` function that calls `createSwapChain` and all
of the creation functions for the objects that depend on the swap chain or the
window size.

```c++
void recreateSwapChain() {
    vkDeviceWaitIdle(device);

    createSwapChain();
    createImageViews();
    createRenderPass();
    createGraphicsPipeline();
    createFramebuffers();
    createCommandBuffers();
}
```

We first call `vkDeviceWaitIdle`, because just like in the last chapter, we
shouldn't touch resources that may still be in use. Obviously, the first thing
we'll have to do is recreate the swap chain itself. The image views need to be
recreated because they are based directly on the swap chain images. The render
pass needs to be recreated because it depends on the format of the swap chain
images. It is rare for the swap chain image format to change during an operation
like a window resize, but it should still be handled. Viewport and scissor
rectangle size is specified during graphics pipeline creation, so the pipeline
also needs to be rebuilt. It is possible to avoid this by using dynamic state
for the viewports and scissor rectangles. Finally, the framebuffers and command
buffers also directly depend on the swap chain images.

To make sure that the old versions of these objects are cleaned up before
recreating them, we should move some of the cleanup code to a separate function
that we can call from the `recreateSwapChain` function. Let's call it
`cleanupSwapChain`:

```c++
void cleanupSwapChain() {

}

void recreateSwapChain() {
    vkDeviceWaitIdle(device);

    cleanupSwapChain();

    createSwapChain();
    createImageViews();
    createRenderPass();
    createGraphicsPipeline();
    createFramebuffers();
    createCommandBuffers();
}
```

we'll move the cleanup code of all objects that are recreated as part of a swap
chain refresh from `cleanup` to `cleanupSwapChain`:

```c++
void cleanupSwapChain() {
    for (size_t i = 0; i < swapChainFramebuffers.size(); i++) {
        vkDestroyFramebuffer(device, swapChainFramebuffers[i], nullptr);
    }

    vkFreeCommandBuffers(device, commandPool, static_cast<uint32_t>(commandBuffers.size()), commandBuffers.data());

    vkDestroyPipeline(device, graphicsPipeline, nullptr);
    vkDestroyPipelineLayout(device, pipelineLayout, nullptr);
    vkDestroyRenderPass(device, renderPass, nullptr);

    for (size_t i = 0; i < swapChainImageViews.size(); i++) {
        vkDestroyImageView(device, swapChainImageViews[i], nullptr);
    }

    vkDestroySwapchainKHR(device, swapChain, nullptr);
}

void cleanup() {
    cleanupSwapChain();

    for (size_t i = 0; i < MAX_FRAMES_IN_FLIGHT; i++) {
        vkDestroySemaphore(device, renderFinishedSemaphores[i], nullptr);
        vkDestroySemaphore(device, imageAvailableSemaphores[i], nullptr);
        vkDestroyFence(device, inFlightFences[i], nullptr);
    }

    vkDestroyCommandPool(device, commandPool, nullptr);

    vkDestroyDevice(device, nullptr);

    if (enableValidationLayers) {
        DestroyDebugUtilsMessengerEXT(instance, debugMessenger, nullptr);
    }

    vkDestroySurfaceKHR(instance, surface, nullptr);
    vkDestroyInstance(instance, nullptr);

    glfwDestroyWindow(window);

    glfwTerminate();
}
```

We could recreate the command pool from scratch, but that is rather wasteful.
Instead I've opted to clean up the existing command buffers with the
`vkFreeCommandBuffers` function. This way we can reuse the existing pool to
allocate the new command buffers.

To handle window resizes properly, we also need to query the current size of the framebuffer to make sure that the swap chain images have the (new) right size. To do that change the `chooseSwapExtent` function to take the actual size into account:

```c++
VkExtent2D chooseSwapExtent(const VkSurfaceCapabilitiesKHR& capabilities) {
    if (capabilities.currentExtent.width != UINT32_MAX) {
        return capabilities.currentExtent;
    } else {
        int width, height;
        glfwGetFramebufferSize(window, &width, &height);

        VkExtent2D actualExtent = {
            static_cast<uint32_t>(width),
            static_cast<uint32_t>(height)
        };

        ...
    }
}
```

That's all it takes to recreate the swap chain! However, the disadvantage of
this approach is that we need to stop all rendering before creating the new swap
chain. It is possible to create a new swap chain while drawing commands on an
image from the old swap chain are still in-flight. You need to pass the previous
swap chain to the `oldSwapChain` field in the `VkSwapchainCreateInfoKHR` struct
and destroy the old swap chain as soon as you've finished using it.

## Suboptimal or out-of-date swap chain

Now we just need to figure out when swap chain recreation is necessary and call
our new `recreateSwapChain` function. Luckily, Vulkan will usually just tell us that the swap chain is no longer adequate during presentation. The `vkAcquireNextImageKHR` and
`vkQueuePresentKHR` functions can return the following special values to
indicate this.

* `VK_ERROR_OUT_OF_DATE_KHR`: The swap chain has become incompatible with the
surface and can no longer be used for rendering. Usually happens after a window resize.
* `VK_SUBOPTIMAL_KHR`: The swap chain can still be used to successfully present
to the surface, but the surface properties are no longer matched exactly.

```c++
VkResult result = vkAcquireNextImageKHR(device, swapChain, UINT64_MAX, imageAvailableSemaphores[currentFrame], VK_NULL_HANDLE, &imageIndex);

if (result == VK_ERROR_OUT_OF_DATE_KHR) {
    recreateSwapChain();
    return;
} else if (result != VK_SUCCESS && result != VK_SUBOPTIMAL_KHR) {
    throw std::runtime_error("failed to acquire swap chain image!");
}
```

If the swap chain turns out to be out of date when attempting to acquire an
image, then it is no longer possible to present to it. Therefore we should
immediately recreate the swap chain and try again in the next `drawFrame` call.

However, if we abort drawing at this point then the fence will never have
been submitted with `vkQueueSubmit` and it'll be in an unexpected state when we
try to wait for it later on. We could recreate the fences as part of swap chain
recreation, but it's easier to move the `vkResetFences` call:

```c++
vkResetFences(device, 1, &inFlightFences[currentFrame]);

if (vkQueueSubmit(graphicsQueue, 1, &submitInfo, inFlightFences[currentFrame]) != VK_SUCCESS) {
    throw std::runtime_error("failed to submit draw command buffer!");
}
```

You could also decide to do that if the swap chain is suboptimal, but I've
chosen to proceed anyway in that case because we've already acquired an image.
Both `VK_SUCCESS` and `VK_SUBOPTIMAL_KHR` are considered "success" return codes.

```c++
result = vkQueuePresentKHR(presentQueue, &presentInfo);

if (result == VK_ERROR_OUT_OF_DATE_KHR || result == VK_SUBOPTIMAL_KHR) {
    recreateSwapChain();
} else if (result != VK_SUCCESS) {
    throw std::runtime_error("failed to present swap chain image!");
}

currentFrame = (currentFrame + 1) % MAX_FRAMES_IN_FLIGHT;
```

The `vkQueuePresentKHR` function returns the same values with the same meaning.
In this case we will also recreate the swap chain if it is suboptimal, because
we want the best possible result.

## Handling resizes explicitly

Although many drivers and platforms trigger `VK_ERROR_OUT_OF_DATE_KHR` automatically after a window resize, it is not guaranteed to happen. That's why we'll add some extra code to also handle resizes explicitly. First add a new member variable that flags that a resize has happened:

```c++
std::vector<VkFence> inFlightFences;
size_t currentFrame = 0;

bool framebufferResized = false;
```

The `drawFrame` function should then be modified to also check for this flag:

```c++
if (result == VK_ERROR_OUT_OF_DATE_KHR || result == VK_SUBOPTIMAL_KHR || framebufferResized) {
    framebufferResized = false;
    recreateSwapChain();
} else if (result != VK_SUCCESS) {
    ...
}
```

It is important to do this after `vkQueuePresentKHR` to ensure that the semaphores are in a consistent state, otherwise a signalled semaphore may never be properly waited upon. Now to actually detect resizes we can use the `glfwSetFramebufferSizeCallback` function in the GLFW framework to set up a callback:

```c++
void initWindow() {
    glfwInit();

    glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);

    window = glfwCreateWindow(WIDTH, HEIGHT, "Vulkan", nullptr, nullptr);
    glfwSetFramebufferSizeCallback(window, framebufferResizeCallback);
}

static void framebufferResizeCallback(GLFWwindow* window, int width, int height) {

}
```

The reason that we're creating a `static` function as a callback is because GLFW does not know how to properly call a member function with the right `this` pointer to our `HelloTriangleApplication` instance.

However, we do get a reference to the `GLFWwindow` in the callback and there is another GLFW function that allows you to store an arbitrary pointer inside of it: `glfwSetWindowUserPointer`:

```c++
window = glfwCreateWindow(WIDTH, HEIGHT, "Vulkan", nullptr, nullptr);
glfwSetWindowUserPointer(window, this);
glfwSetFramebufferSizeCallback(window, framebufferResizeCallback);
```

This value can now be retrieved from within the callback with `glfwGetWindowUserPointer` to properly set the flag:

```c++
static void framebufferResizeCallback(GLFWwindow* window, int width, int height) {
    auto app = reinterpret_cast<HelloTriangleApplication*>(glfwGetWindowUserPointer(window));
    app->framebufferResized = true;
}
```

Now try to run the program and resize the window to see if the framebuffer is indeed resized properly with the window.

## Handling minimization

There is another case where a swap chain may become out of data and that is a special kind of window resizing: window minimization. This case is special because it will result in a frame buffer size of `0`. In this tutorial we will handle that by pausing until the window is in the foreground again by extending the `recreateSwapChain` function:

```c++
void recreateSwapChain() {
    int width = 0, height = 0;
    while (width == 0 || height == 0) {
        glfwGetFramebufferSize(window, &width, &height);
        glfwWaitEvents();
    }

    vkDeviceWaitIdle(device);

    ...
}
```

Congratulations, you've now finished your very first well-behaved Vulkan
program! In the next chapter we're going to get rid of the hardcoded vertices in
the vertex shader and actually use a vertex buffer.

[C++ code](/code/16_swap_chain_recreation.cpp) /
[Vertex shader](/code/09_shader_base.vert) /
[Fragment shader](/code/09_shader_base.frag)
