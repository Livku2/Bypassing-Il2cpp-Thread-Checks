# Bypassing-Il2cpp-Thread-Checks

Sometimes when you try to call an il2cpp symbol such as ```il2cpp_class_from_name```, it may return nullptr, this is due to one of three possible issues. Luckily, the first two solutions are very similar.

### Requirements
You will need a hooking library, for this example I will use dobby, but you can use any you want.

## Issue 1

The first issue may be that il2cpp_domain_get_assemblies returns nullptr.

il2cpp_class_from_name requires three params, image, namespace, and name.
The way you may normally get the image is through using il2cpp_domain_get_assemblies and sorting through each one until they have the correct name, like this:

```cpp
void* GetImageByName(const char *image) {
    size_t size;
    void **assemblies = il2cpp_domain_get_assemblies(il2cpp_domain_get(), &size);
    for(int i = 0; i < size; ++i)
    {
        void *img = (void *)il2cpp_assembly_get_image(assemblies[i]);

        const char *img_name = il2cpp_image_get_name(img);

        if(strcmp(img_name, image) == 0)
        {
            return img;
        }
    }
    return nullptr;
}
```

If il2cpp_domain_get_assemblies returns nullptr and a size of 0, then GetImageByName will return nullptr as well. This may cause il2cpp_class_from_name to crash due to the image being invalid

### Solution

We will hook class from name to get the images

First, make a vector for images

```cpp
std::vector<const void*> images;
```

Then create the new method

```cpp

void* (*il2cpp_class_from_name)(const void* image, const char* ns, const char* name);

void* new_il2cpp_class_from_name(const void* image, const char* ns, const char* name) {

    //Check if the vector contains the image, and if it doesn't, add the image to it
    if (!std::count(images.begin(), images.end(), image)) {
        images.push_back(image);
    }

    return orig_il2cpp_class_from_name(image, ns, name);
}
```

Then get the il2cpp_class_from_name address and hook it
```cpp

        void* handle = dlopen("libil2cpp.so", RTLD_LAZY);
        if (handle) {
            void* target_func = dlsym(handle, "il2cpp_class_from_name");
            if (target_func) {
                //Use any hooking library of your choice
                DobbyHook(target_func, reinterpret_cast<dobby_dummy_func_t>(new_il2cpp_class_from_name), reinterpret_cast<dobby_dummy_func_t*>(&il2cpp_class_from_name));

                LOGI("Hooked il2cpp_class_from_name successfully!");
            } else {
                LOGI("Failed to find il2cpp_class_from_name symbol");
            }
        } else {
            LOGI("Failed to load libil2cpp.so");
        }
```

now you can filter through all your images with something like this

```cpp

void* GetImageByName(const char *image) {
    auto size = images.size();
    for(int i = 0; i < size; ++i)
    {
        void *img = const_cast<void *>(images[i]);

        const char *img_name = il2cpp_image_get_name(img);

        if(strcmp(img_name, image) == 0)
        {
            return img;
        }
    }
    return nullptr;
}

```




## Issue 2

Some games implement thread checks in their il2cpp symbols.

We can see if a symbols has a thread check in it by opening the libil2cpp.so file in IDA. I will use il2cpp_class_from_name for this example.

1. Open the libil2cpp.so file up in ida and wait for it to load.
2. When it loads, go to the functions window and sort by function name.
3. Find il2cpp_class_from_name and click on it

You may see something like this

![image](https://github.com/user-attachments/assets/acf7bef1-435d-4cd7-a303-3793bdeea3ab)

We can see the function calling l2cpp_thread_current_0

This confirms that the symbol has a thread check in it.

### Solution

Create a class vector

```cpp
std::vector<const char*> classes;
```

Then create the hook method

you can try calling the original method, getting that class, and then using it for hooking, then returning

```cpp

void* (*il2cpp_class_from_name)(const void* image, const char* ns, const char* name);

void* new_il2cpp_class_from_name(const void* image, const char* ns, const char* name) {

    auto klass = il2cpp_class_from_name(image, ns, name);

    //Check if the vector contains the classes, and if it doesn't, add the classe to it so we don't call the hooks twice
    if (!std::count(classes.begin(), classes.end(), name)) {
        classes.push_back(name);
        
        //use klass for hooking here
    }

    //you may also try calling your own method, because as long as they are called within the il2cpp thread, the il2cpp functions will work
    if(doOnce){
      doOnce = true;
      MySetupMethod();
    }


    return klass;
}
```

Then get the il2cpp_class_from_name address and hook it
```cpp

        void* handle = dlopen("libil2cpp.so", RTLD_LAZY);
        if (handle) {
            void* target_func = dlsym(handle, "il2cpp_class_from_name");
            if (target_func) {
                //Use any hooking library of your choice
                DobbyHook(target_func, reinterpret_cast<dobby_dummy_func_t>(new_il2cpp_class_from_name), reinterpret_cast<dobby_dummy_func_t*>(&il2cpp_class_from_name));

                LOGI("Hooked il2cpp_class_from_name successfully!");
            } else {
                LOGI("Failed to find il2cpp_class_from_name symbol");
            }
        } else {
            LOGI("Failed to load libil2cpp.so");
        }
```

## Issue 3

This is very unlikely, but some games obfuscate their il2cpp symbols. You will have to search on your own for the correct functions.


