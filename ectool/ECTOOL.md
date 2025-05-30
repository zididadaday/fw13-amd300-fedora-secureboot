## ectool

Compiling ectool

```
git clone https://github.com/DHowett/ectool.git ec
cd ec
mkdir _build
cd _build
CC=clang CXX=clang++ cmake -GNinja ..
cmake --build .
```

Checking version `sudo ./ectool version`

```
RO version:    lilac-3.0.3-413f018
RW version:    lilac-3.0.3-413f018
Firmware copy: RO
Build info:    lilac-3.0.3-413f018 2025-03-06 05:45:28 marigold2@ip-172-26-3-226
Tool version:  0.0.1-isolate May 30 2025 none
```

`https://github.com/DHowett/ectool/tree/main`