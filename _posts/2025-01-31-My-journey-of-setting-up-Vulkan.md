---
title: "My journey of setting up Vulkan"
date: 2025-01-31
---
Vulkan is notorious for its difficulty and extremely steep learning curve, and most of it stems from the fact that you need thousands of lines of boilerplate in order to get the simplest of things working. But there are numerous libraries that are designed to make our work easier to make Vulkan more accessible.

# vk-bootstrap and glfw
[vk-bootstrap](https://github.com/charles-lunarg/vk-bootstrap) , can help streamline the process of many tedious tasks that require hours of our attention, into something trivial. It handles the one time tasks we have to endure during our journey of graphics programming such as 
- Instance creation
- Physical Device selection
- Device creation
- Getting queues
- Swapchain creation

It does not create the window on its own, so we use [glfw](https://github.com/glfw/glfw) to do that and pass the window to vk-bootstrap. The setup and the usage of the above libraries can be done as follows.

1. Create a new CMake project. It is recommended to use Visual Studio.
2. Install the Vulkan SDK and GLFW binaries. (https://vulkan-tutorial.com/Development_environment)
3. Initialize a git project using the command `git init`.
4. Add the above libraries and glm for math using these commands.
   ```
   git submodule add https://github.com/charles-lunarg/vk-bootstrap
   git submodule add https://github.com/glfw/glfw.git
   git submodule add https://github.com/g-truc/glm.git
   ```
5. Create a `CMakeFile.txt` in the root of the project and add the following content.
   ```cmake
    cmake_minimum_required (VERSION 3.8)

    if (POLICY CMP0141)
      cmake_policy(SET CMP0141 NEW)
      set(CMAKE_MSVC_DEBUG_INFORMATION_FORMAT "$<IF:$<AND:$<C_COMPILER_ID:MSVC>,$<CXX_COMPILER_ID:MSVC>>,$<$<CONFIG:Debug,RelWithDebInfo>:EditAndContinue>,$<$<CONFIG:Debug,RelWithDebInfo>:ProgramDatabase>>")
    endif()
    
    project ("VulkanTemplate")
    
    find_package(Vulkan REQUIRED)
    
    add_subdirectory(vk-bootstrap)
    add_subdirectory(glfw)
    add_subdirectory(glm)
    
    
    add_executable (VulkanTemplate "VulkanTemplate.cpp" "VulkanTemplate.h")
    
    target_link_libraries(VulkanTemplate Vulkan::Vulkan vk-bootstrap::vk-bootstrap glfw glm)
    
    if (CMAKE_VERSION VERSION_GREATER 3.12)
      set_property(TARGET VulkanTemplate PROPERTY CXX_STANDARD 23)
    endif()

   ```
6. Paste the following in the main source file. It will create the window as well as the instance, physical device, logical device and the surface.
   ```c++
    #define VK_USE_PLATFORM_WIN32_KHR
    #define GLFW_INCLUDE_VULKAN
    #include <GLFW/glfw3.h>
    #define GLFW_EXPOSE_NATIVE_WIN32
    #include <GLFW/glfw3native.h>
    #include <vulkan/vulkan.h>
    
    #define GLM_FORCE_RADIANS
    #define GLM_FORCE_DEPTH_ZERO_TO_ONE
    #include <glm/vec4.hpp>
    #include <glm/mat4x4.hpp>
    
    #include <iostream>
    
    #include "VkBootstrap.h"
    
    int main() {
        glfwInit();
    
        glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);
        GLFWwindow* window = glfwCreateWindow(800, 600, "Vulkan window", nullptr, nullptr);
    
        uint32_t extensionCount = 0;
        vkEnumerateInstanceExtensionProperties(nullptr, &extensionCount, nullptr);
    
        std::cout << extensionCount << " extensions supported\n";
    
        VkSurfaceKHR surface;
        VkWin32SurfaceCreateInfoKHR createInfo{};
        createInfo.sType = VK_STRUCTURE_TYPE_WIN32_SURFACE_CREATE_INFO_KHR;
        createInfo.hwnd = glfwGetWin32Window(window);
        createInfo.hinstance = GetModuleHandle(nullptr);
    
        vkb::InstanceBuilder builder;
        auto inst_ret = builder.set_app_name("Example Vulkan Application")
            .request_validation_layers()
            .use_default_debug_messenger()
            .build();
        if (!inst_ret) {
            std::cerr << "Failed to create Vulkan instance. Error: " << inst_ret.error().message() << "\n";
            return 1;
        }
        vkb::Instance vkb_inst = inst_ret.value();
    
        if (vkCreateWin32SurfaceKHR(vkb_inst.instance, &createInfo, nullptr, &surface) != VK_SUCCESS) {
            throw std::runtime_error("failed to create window surface!");
        }
    
        vkb::PhysicalDeviceSelector selector{ vkb_inst };
        auto phys_ret = selector.set_surface(surface)
            .set_minimum_version(1, 1) // require a vulkan 1.1 capable device
            .require_dedicated_transfer_queue()
            .select();
        if (!phys_ret) {
            std::cerr << "Failed to select Vulkan Physical Device. Error: " << phys_ret.error().message() << "\n";
            return 1;
        }
    
        vkb::DeviceBuilder device_builder{ phys_ret.value() };
        // automatically propagate needed data from instance & physical device
        auto dev_ret = device_builder.build();
        if (!dev_ret) {
            std::cerr << "Failed to create Vulkan device. Error: " << dev_ret.error().message() << "\n";
            return 1;
        }
        vkb::Device vkb_device = dev_ret.value();
    
        // Get the VkDevice handle used in the rest of a vulkan application
        VkDevice device = vkb_device.device;
    
        // Get the graphics queue with a helper function
        auto graphics_queue_ret = vkb_device.get_queue(vkb::QueueType::graphics);
        if (!graphics_queue_ret) {
            std::cerr << "Failed to get graphics queue. Error: " << graphics_queue_ret.error().message() << "\n";
            return 1;
        }
        VkQueue graphics_queue = graphics_queue_ret.value();
    
        while (!glfwWindowShouldClose(window)) {
            glfwPollEvents();
        }
    
        glfwDestroyWindow(window);
    
        glfwTerminate();
    }
   ```
7. Run the program. We have now created the window with minimak effort.
