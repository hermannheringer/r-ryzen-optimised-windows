# Accelerating R with AMD AOCL BLAS on Windows

## Why R, not Python?

It's just a personal choice. R fits the way I like to work through the whole process: getting the information, cleaning up mistakes, organizing it, checking if it's trustworthy, using it to make predictions, and turning it into something useful.

For dense numerical linear algebra on laptops and workstations, R gives me a clean, batteries‑included workflow that make performance analysis straightforward. When the bottleneck is matrix algebra rather than web frameworks, R exposes BLAS/LAPACK directly while staying close to the statistical models and diagnostics I care about.

leaving the stock BLAS in place means wasting a lot of available throughput.

A simple SVD benchmark on a tall matrix makes the point.

Native R BLAS:

```r
m <- 100000
n <- 2000
A <- matrix(runif(m * n), m, n)
system.time(S <- svd(A, nu = 0, nv = 0))
#   user  system elapsed 
# 259.1     1.8   261.8
```

Using AMD Optimizing CPU Libraries (AOCL) for BLAS:

```r
m <- 100000
n <- 2000
A <- matrix(runif(m * n), m, n)
system.time(S <- svd(A, nu = 0, nv = 0))
#   user  system elapsed 
# 140.5    22.6    12.8
```

The rest of this document walks through wiring AOCL BLIS into R on Windows, step by step.

> **Note**  
> This is “hard mode”. You are replacing core numerical libraries. Keep backups of the original DLLs and be ready to revert if something breaks.

---

## 1. Prerequisites

1. **64‑bit R installation**

   This guide assumes R 4.6.1 installed at:

   ```text
   C:\Program Files\R\R-4.6.1\bin\x64
   ```

2. **AMD AOCL for Windows**

   Install AOCL for Windows from AMD’s developer site ( https://www.amd.com/en/developer/aocl.html#downloads ). AOCL provides AOCL‑BLAS and AOCL‑LAPACK; on Windows these come as DLLs built with Clang.

3. **Administrator rights**

   You need permission to write under `C:\Program Files` and install runtimes.

---

## 2. Locate AOCL BLIS DLLs

After installing AOCL, locate the BLIS. On a default setup they live under:

```text
C:\Program Files\AMD\AOCL-Windows
```

within `amd-blis`

Use PowerShell to confirm:

```powershell
Get-ChildItem -Path "C:\Program Files\AMD\AOCL-Windows" -Recurse `
  -Include "AOCL-LibBlis-Win-*.dll"
```

You should see files like:

```text
AOCL-LibBlis-Win-dll.dll
AOCL-LibBlis-Win-MT-dll.dll
```

Make sure you pick the **LP64, x64, ‑MT** variants (no `i386` in the name and `MT` suffix). These match how R is built.

---

## 3. Back up the original R BLAS DLLs

Before touching anything, back up the stock BLAS DLLs.

From an elevated PowerShell:

```powershell
Set-Location "C:\Program Files\R\R-4.6.1\bin\x64"

Copy-Item .\Rblas.dll   .\Rblas.dll.orig
```

If you need to revert later, restore these backups.

---

## 4. Drop AOCL BLIS in as Rblas

The BLAS layer is where most matrix time is spent and it is the easiest layer to swap.

Copy AOCL BLIS into the R `bin\x64` directory and rename it to `Rblas.dll`:

```powershell
Copy-Item "C:\Program Files\AMD\AOCL-Windows\amd-blis\lib\LP64\AOCL-LibBlis-Win-MT-dll.dll" `
          "C:\Program Files\R\R-4.6.1\bin\x64\Rblas.dll"
```

Ensure there is a single `Rblas.dll` in that directory and that its size matches the AOCL BLIS DLL.

At this point, R will use AOCL BLIS for its BLAS routines, which is enough to reach the singular value decomposition speed‑up shown earlier.

---

The BLAS swap alone already delivers most of the practical speed‑up for typical workloads.

---

## 5. Provide an OpenMP runtime

AOCL BLIS rely on an OpenMP runtime. On Windows, AOCL expects an Intel‑style `libiomp5md.dll`, but LLVM’s `libomp140.x86_64.dll` works fine when renamed.

First locate a suitable `libomp` DLL:

```powershell
Get-ChildItem -Path C:\ -Include *libomp* -File -Recurse -ErrorAction SilentlyContinue
```

On a typical Visual Studio / Windows 11 install you will see, among others:

```text
C:\Windows\System32\libomp140.x86_64.dll
C:\Windows\System32\libomp140d.x86_64.dll
C:\Windows\SysWOW64\libomp140.i386.dll
C:\Windows\SysWOW64\libomp140d.i386.dll
```

Pick the **64‑bit release build**:

- `C:\Windows\System32\libomp140.x86_64.dll` (no `d` means debug, no `i386`).

Then copy it into the R `bin\x64` folder and rename:

```powershell
Set-Location 'C:\Program Files\R\R-4.6.1\bin\x64'

Copy-Item 'C:\Windows\System32\libomp140.x86_64.dll' -Destination .\libiomp5md.dll
```

This leaves the system copy in `System32` intact and gives AOCL BLIS an OpenMP runtime located alongside `Rblas.dll`.

---

## 6. Verify from inside R

Start R and inspect the math libraries in use:

```r
sessionInfo()
```

You should see something like:

```text
Matrix products: default
  LAPACK version 3.12.1
```

Then re‑run the SVD benchmark:

```r
m <- 100000
n <- 2000
A <- matrix(runif(m * n), m, n)

system.time(S <- svd(A, nu = 0, nv = 0))
```

On the Ryzen 7 5800H used here, elapsed time drops from ~262 seconds with the native BLAS to ~13 seconds with AOCL BLAS, with CPU utilisation reflecting multi‑threaded execution.

---

## 7. Reverting to the stock configuration

To go back to the original R math stack:

```powershell
Set-Location "C:\Program Files\R\R-4.6.1\bin\x64"

Move-Item .\Rblas.dll.orig   .\Rblas.dll   -Force
```

Remove or rename `libiomp5md.dll` if needed.

The idea is to keep AOCL BLAS as a drop‑in performance upgrade while preserving a straightforward rollback path.

Have fun!