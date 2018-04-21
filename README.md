# TinyDotNet
This project documents my research into creating tiny .NET executables. At the moment I can successfully make small programs (e.g. Hello World) with a file size of 1,536 bytes (actually using 1104 bytes of this, with some slack in the PE headers too), and am experimenting with pushing this down to 1,024 bytes.

## Motivation and Goals

My motivation for this project is the idea of creating a 4K [demo](https://en.wikipedia.org/wiki/Demoscene) in .NET, i.e. a program that displays realtime graphics and plays music, using no more than 4096 bytes. I'm not so great at the music and graphics side of things but I like the idea of pushing the limits of the .NET platform, especially since no suitable compression tools exist for 4K size targets.

I'm primarily targeting .NET Core, since the Core CLR is open-source and cross-platform. This makes the challenge and result more interesting - a fully cross-platform demo would be quite fun to make - but also more achievable than targeting standard .NET, as I can debug the CLR's assembly loader.

The ultimate goal is to build a generic compressor tool for .NET Core applications, targeting the 4K file size.

## Shrinking the Assembly

Assemblies for .NET Core are DLL files which are hosted with the `dotnet` tool (and some others). The structure is just like a regular .NET executable - it's a PE file with a .NET metadata directory. We can use many of the usual PE shrinking tricks, alongside some .NET specific ones.

### Metadata Names

By default the metadata has names for classes, variables, etc. which we can shorten down to a single character each. This saves us a whole load of string data we don't want.

### Assembly Attributes

The compiler puts a lot of informational attributes onto the assembly object. These have some pretty long string names and require a `CustomAttribute` table entry each. These can be trivially removed manually with Reflexil and on average saves around 600 bytes.

### DOS Stub

The DOS stub obviously isn't needed. We can get rid of that entirely and have the NT/PE header directly following the DOS header (`e_lfanew = sizeof(DOS_Header)+1`). This saves a bunch of bytes from the start of the file, although it doesn't matter a whole lot since the initial section must align to 0x200.

### Entry Point

The entry point begins with an x86 indirect jump to the address at `0x00402000`, which contains `0x000027E8`, which is the IAT address for `mscoree.dll!_CoreExeMain`. This EP is never executed if you load the DLL with `dotnet`. I have verified this by replacing the EP with garbage opcodes and `int3` breakpoints. This means we can point the EP to any valid RVA, and remove the entry point instructions.

### Data Directories

There are Import, Resource, Relocation, Debug, IAT, and .NET MetaData directories by default. The `NumberOfRVAsAndSizes` field of the PE header must be set to 16 because otherwise the loader can't find the .NET MetaData directory. This means no directory folding tricks. The debug table can be completely removed, as can the import table (it isn't actually used by the loader!), which means the IAT and relocation table can go too.

### Sections

By default there are three sections: .text, .rsrc, and .reloc for the metadata and code, resources, and relocation table respectively. We can completely delete the resource section as it only contains the manifest and version data, which isn't required. We can also delete the relocation section as it isn't needed once we get rid of the import table. Having only one section is critical because each additional section must, due to the alignment, add at least 512 bytes. By rebuilding the PE with 512-byte alignment, this cut down executable gets us to the 1,536 byte mark for a small Hello World.

## Future Improvement

- Rename the assembly from "tinype" to "t". Saves 5 bytes.
- Can stuff main code into .ctor and run it directly from there. Means we get rid of the IL from .ctor, one method entry, and the method name. Saves 21 bytes.
- `#GUID` table entry was "removed" by shifting it to the end of the stream table and decrementing the stream count, but the table entry is still there. Shifting things up saves about 16 bytes but so far I've not had much luck with getting this working. I think it messes up an offset somewhere, or maybe a length check, but I can't really tell where.

## Negative Results

Here's where all my failed attempts go.

### Native import shenanigans

Adding a native DLL (e.g. user32.dll) to the import table doesn't work. The main module loads fine, but the dependency loader tries to load the native DLL as a .NET Core assembly.

### Section at offset zero

Creating a PE with a single section at offset 0, with no import directory, IAT, relocations, etc. got me a different error - the PE load sort of works, but CoreCLR's sanity checks fail:

```
Assert failure(PID 21220 [0x000052e4], Thread: 12436 [0x3094]): Precondition failure: FAILED: addressStart >= previousAddressEnd && (offsetSize == 0 || offsetStart >= previousOffsetEnd)
         FAILED: CheckSection(currentAddress, section->VirtualAddress, section->Misc.VirtualSize, currentOffset, section->PointerToRawData, section->SizeOfRawData)
                d:\code\coreclr\src\utilcode\pedecoder.cpp, line: 363
         FAILED: CheckNTHeaders()
                d:\code\coreclr\src\inc\pedecoder.inl, line: 713

CORECLR! CHECK::Trigger + 0x275 (0x00007ffb`6089d4d5)
CORECLR! PEDecoder::HasCorHeader + 0x302 (0x00007ffb`6091a922)
CORECLR! PEDecoder::IsNativeMachineFormat + 0x68 (0x00007ffb`60e97d48)
CORECLR! MappedImageLayout::MappedImageLayout + 0x59C (0x00007ffb`610e8bfc)
CORECLR! PEImageLayout::Map + 0x45D (0x00007ffb`610ebb1d)
CORECLR! PEImage::CreateLayoutMapped + 0x2E9 (0x00007ffb`60b56c99)
CORECLR! PEImage::GetLayoutInternal + 0x306 (0x00007ffb`60b58616)
CORECLR! PEImage::GetLayout + 0xD9 (0x00007ffb`60b582b9)
CORECLR! BinderAcquireImport + 0x105 (0x00007ffb`60e2d045)
CORECLR! BINDER_SPACE::AssemblyBinder::GetAssembly + 0x2B5 (0x00007ffb`614fc185)
    File: d:\code\coreclr\src\utilcode\pedecoder.cpp Line: 427
    Image: D:\Code\coreclr\bin\Product\Windows_NT.x64.Debug\CoreRun.exe
```

We can see here that the CLR is checking to see if the section's raw base address is greater than the current read pointer for the headers. This prevents us from crushing the PE headers and .NET metadata together. A shame really because that'd get us to the 1,024 byte mark easily.
