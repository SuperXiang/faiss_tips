# Build
This is a step-by-step instruction for building faiss from source. We assume:
- x86 architecture
- CPU
- Ubuntu 20.04 
- miniconda for python environment
- Intel MKL (we can install it simply by apt for Ubuntu 20.04+)
- AVX2

Official documents:
- [Official installation guide](https://github.com/facebookresearch/faiss/blob/master/INSTALL.md)
- [Official wiki](https://github.com/facebookresearch/faiss/wiki/Installing-Faiss)
- [Official conda config](https://github.com/facebookresearch/faiss/tree/master/conda)

## [tl;dr](build.sh)

## Prerequisite

### g++ and swig
```bash
sudo apt install -y build-essential swig
```

### Intel MKL
Installing Intel MKL has been extremely hard. Fortunately, for Ubuntu 20.04 or higher, we can install it simply by `apt install`.
```bash
sudo apt install -y intel-mkl
```
The official wiki introduces [the way to use MKL inside the anaconda](https://github.com/facebookresearch/faiss/wiki/Installing-Faiss). I've tried it dozens of times, and it doesn't work. If anyone can make it work, please send me an issue/PR.

If you cannot install intel-mkl, you can use open-blas by `sudo apt install -y libopenblas-dev`


### cmake
Currently, cmake from apt is old (3.16). There are three options to install new cmake.
- Build from source
- Install by snap. This is the easiest.
    ```bash
    sudo snap install cmake --classic
    ```
- If you've installed conda, you can install cmake by conda.
    ```bash
    conda install -c anaconda cmake 
    ```

### miniconda
We will use miniconda for python. See [this](https://conda.io/projects/conda/en/latest/user-guide/install/macos.html#install-macos-silent) for the instruction of the silent installation.
```bash
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh -O ~/miniconda.sh
bash ~/miniconda.sh -b -p $HOME/miniconda
```
Then activate the miniconda
```bash
echo 'export PATH="$HOME/miniconda/bin:$PATH"' >> $HOME/.bashrc
source $HOME/.bashrc
```
Install required packages
```bash
conda update conda --yes
conda update --all --yes
conda install numpy --yes
```
Make sure your python path works
```bash
which python    # /home/ubuntu/miniconda/bin/python
```


## Build c++
Clone the repo.
```bash
git clone https://github.com/facebookresearch/faiss.git
cd faiss
```
Run cmake. See the [official instruction](https://github.com/facebookresearch/faiss/blob/master/INSTALL.md#step-1-invoking-cmake) for the explanation of each option

```bash
cmake -B build \
    -DBUILD_SHARED_LIBS=ON \
    -DBUILD_TESTING=ON \
    -DFAISS_OPT_LEVEL=avx2 \
    -DFAISS_ENABLE_GPU=OFF \
    -DFAISS_ENABLE_PYTHON=OFF \
    -DCMAKE_BUILD_TYPE=Release .
```
This `cmake` creates a `build` directory.
Note that you don't need to specify `-DBLA_VENDOR` and `-DMKL_LIBRARIES`.

In the log message, you will find that MKL is correctly found: `-- Found MKL: /usr/lib/x86_64-linux-gnu/libmkl_intel_lp64.so;/usr/lib/x86_64-linux-gnu/libmkl_sequential.so;/usr/lib/x86_64-linux-gnu/libmkl_core.so;-lpthread;-lm;-ldl`


Then, run make to build the library.
```bash
make -C build -j faiss faiss_avx2
```
This will create `build/faiss/libfaiss.so` and `build/faiss/libfaiss_avx2.so`. I am not well unserstand this point but we need to manually specify `libfaiss_avx2.so`.

Let's check the link information by:
```bash
ldd build/faiss/libfaiss_avx2.so
```
This will show something like:
```bash
        linux-vdso.so.1 (0x00007ffc6dcc7000)
        libmkl_intel_lp64.so => /lib/x86_64-linux-gnu/libmkl_intel_lp64.so (0x00007f4e3cfd1000)
        libmkl_sequential.so => /lib/x86_64-linux-gnu/libmkl_sequential.so (0x00007f4e3b9b9000)
        libmkl_core.so => /lib/x86_64-linux-gnu/libmkl_core.so (0x00007f4e37699000)
        libpthread.so.0 => /lib/x86_64-linux-gnu/libpthread.so.0 (0x00007f4e37676000)
        libgomp.so.1 => /lib/x86_64-linux-gnu/libgomp.so.1 (0x00007f4e37634000)
        libstdc++.so.6 => /lib/x86_64-linux-gnu/libstdc++.so.6 (0x00007f4e37450000)
        libm.so.6 => /lib/x86_64-linux-gnu/libm.so.6 (0x00007f4e37301000)
        libgcc_s.so.1 => /lib/x86_64-linux-gnu/libgcc_s.so.1 (0x00007f4e372e6000)
        libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f4e370f4000)
        /lib64/ld-linux-x86-64.so.2 (0x00007f4e3de2a000)
        libdl.so.2 => /lib/x86_64-linux-gnu/libdl.so.2 (0x00007f4e370ee000)
```
Here, you can see `/lib/x86_64-linux-gnu/libmkl_intel_lp64.so`, etc. This means the faiss is using the Intel MKL that is installed on the system by `apt`.


Then let's test. It seems `make -C build test` doesn't work. So let's try `demo_ivfpq_indexing`

```bash
make -C build demo_ivfpq_indexing
./build/demos/demo_ivfpq_indexing
```
It takes 7 sec for AWS EC2 c5.12xlarge: `[7.298 s] Query results (vector ids, then distances):` 



## Build python
If you make sure that the c++ faiss works, let's move to python.
Let us delete the build directory first and cmake again.
```bash
pwd    # Make sure you are at the root directory of faiss, such as /home/ubuntu/faiss
rm -rf build
```

Let's run cmake.
```
cmake -B build \
    -DBUILD_SHARED_LIBS=ON \
    -DBUILD_TESTING=ON \
    -DFAISS_OPT_LEVEL=avx2 \
    -DFAISS_ENABLE_GPU=OFF \
    -DFAISS_ENABLE_PYTHON=/home/ubuntu/miniconda/bin/python \
    -DCMAKE_BUILD_TYPE=Release .
```
For `-DPython_EXECUTABLE`, write the output of `which python`.

Run make 
```bash
make -C build -j faiss faiss_avx2
```

Then let's build the python module. Run the following.
```bash
make -C build -j swigfaiss swigfaiss_avx2
```

Then let's install it on your python.
```bash
cd build/faiss/python
python setup.py install
```

Finally, you need to specify the PYTHONPATH.
```bash
export PYTHONPATH=/home/ubuntu/faiss/build/faiss/python/build/lib:$PYTHONPATH
```
Please modify the path according to your environment.

Now you can use faiss from python.
Let's check it.
```bash
cd    # Do this. We need to verify that we can use python-faiss from any place
python -c "import faiss, numpy; err = faiss.Kmeans(10, 20).train(numpy.random.rand(1000, 10).astype('float32')); print(err)"
```
You will see something like `483.5049743652344`.

Note that you need to run `export PYTHONPATH= ... ` everytime when you reload the terminal. In order to always activate the python path, please write it on you `.bashrc` manually, or by the following command.

```bash
echo 'export PYTHONPATH=/home/ubuntu/faiss/build/faiss/python/build/lib:$PYTHONPATH' >> $HOME/.bashrc
```

Then close your terminal, open it, and run the above `pyathon -c "import faiss...` again. If it works, that's all.


## Check AVX2 is working or not
Let's check AVX2 is activated or not.

```bash
cd
LD_DEBUG=libs python -c "import faiss" 2>&1 | grep libfaiss.so
```
If you see something, then your AVX2 **is not** activated.


Run the following as well
```bash
cd
LD_DEBUG=libs python -c "import faiss" 2>&1 | grep libfaiss_avx2.so
```
If you see something, then your AVX2 **is** activated.

To actually evaluate the runtime, please save the following as `check.py`
```python
import faiss
import numpy as np
import time

np.random.seed(123)
D = 128
N = 1000
X = np.random.random((N, D)).astype(np.float32)
M = 64
nbits = 4

pq = faiss.IndexPQ(D, M, nbits)
pq.train(X)
pq.add(X)

pq_fast = faiss.IndexPQFastScan(D, M, nbits)
pq_fast.train(X)
pq_fast.add(X)

t0 = time.time()
d1, ids1 = pq.search(x=X[:3], k=5)
t1 = time.time()
print(f"pq: {(t1 - t0) * 1000} msec")

t0 = time.time()
d2, ids2 = pq_fast.search(x=X[:3], k=5)
t1 = time.time()
print(f"pq_fast: {(t1 - t0) * 1000} msec")

assert np.allclose(ids1, ids2)
```

Then run `python check.py`.
If AVX2 is properly activated, pq_fast should be more roughly 10x faster:
```bash
pq: 0.5166530609130859 msec
pq_fast: 0.06580352783203125 msec
```

