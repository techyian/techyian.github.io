---
layout: post
title: Dereference a double pointer in C#
category: coding
description: Learn how dereference a double pointer or "pointer to a pointer" in C#.
tags: [coding, c#, .net]
---

Before I started developing MMALSharp I had never needed to deal with low level memory access or unmanaged resources in C#; 
I had done some C programming whilst at university, but those days were mainly over and my daily programming at work is done 
in a managed environment. But, as MMALSharp is heavily dependent on the C implementation of the MMAL library, things were 
about to change...fast.

Before I knew it, I was analysing large amounts of C code, refreshing my mind on pointers 
(can I not just use a class and forget this ever happened?) and low level memory management; it was a lot to take in at first, 
and the first hurdle appeared when I came across this structure:

```csharp
typedef struct MMAL_COMPONENT_T
{
   // Removed irrelevant areas.
   uint32_t    input_num;   /**< Number of input ports */
   MMAL_PORT_T **input;     /**< Array of input ports */
} MMAL_COMPONENT_T;
```

Without drifting away from the focus of this article too much, this struct represents a component in MMAL, 
and each component has a number of ports, nothing too complicated at high level. However, I had never come across a pointer 
being declared like this before `MMAL_PORT_T **input;`, so a quick Google explained these were "pointers to pointers" or a 
"double pointer" and is essentially a way of declaring an "array of pointers in C". Now I had my work cut out as I needed to 
dereference these "double pointers" to get each individual port from the component. After many attempts, I finally figured out what 
was needed and the solution can be seen below.

For clarity, here is the C# equivalent struct, again all the irrelevant code has been removed:

```csharp
[StructLayout(LayoutKind.Sequential)]
public unsafe struct MMAL_COMPONENT_T
{
    private uint inputNum;
    private MMAL_PORT_T** input;
    
    public uint InputNum => this.inputNum;

    public MMAL_PORT_T** Input => this.input;

    public MMAL_COMPONENT_T(uint inputNum,
                            MMAL_PORT_T** input)
    {
        this.inputNum = inputNum;
        this.input = input;
    }
}
```

Next, you need to loop around the number of pointers you will be expecting to be present, and the dereferencing is done as follows:

```csharp
&(*this.Ptr->Input[i])
```

Where `this.Ptr` is an `IntPtr` to the MMAL component and its `Input` struct member is accessed at index `i`.

I hope that might help someone in a similar situation, and thanks for reading.
