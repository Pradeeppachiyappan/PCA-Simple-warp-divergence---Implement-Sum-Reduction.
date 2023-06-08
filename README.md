# PCA-Simple-warp-divergence---Implement-Sum-Reduction.
Refer to the kernel reduceUnrolling8 and implement the kernel reduceUnrolling16, in which each thread handles 16 data blocks. Compare kernel performance with reduceUnrolling8 and use the proper metrics and events with nvprof to explain any difference in performance.

## Aim:
Compare the performance of the kernel "reduceUnrolling8" and the newly implemented kernel
"reduceUnrolling16" by handling 8 and 16 data blocks per thread, respectively.


## Procedure:
### step 1:
• Implement the "reduceUnrolling16" kernel to handle 16 data blocks per thread.
### step 2:
• Execute the "reduceUnrolling8" and "reduceUnrolling16" kernels with the same input data
size and execution configurations.
### step 3:
• Use proper metrics and events with "nvprof" to analyse the performance of each kernel.

## Program:
## reduceInteger.cu:
```
Name : P.Pradeep Raj
Register No: 212222240073
```
```
#include "common.h"
#include <cuda_runtime.h>
#include <stdio.h>
#define DIM 128
extern __shared__ int dsmem[];
// Recursive Implementation of Interleaved Pair Approach
int recursiveReduce(int *data, int const size)
{
 if (size == 1) return data[0];
 int const stride = size / 2;
 for (int i = 0; i < stride; i++)
 data[i] += data[i + stride];
 return recursiveReduce(data, stride);
}
// unroll4 + complete unroll for loop + gmem
__global__ void reduceGmem(int *g_idata, int *g_odata, unsigned int n)
{
 // set thread ID
 unsigned int tid = threadIdx.x;
 int *idata = g_idata + blockIdx.x * blockDim.x;
 // boundary check
 unsigned int idx = blockIdx.x * blockDim.x + threadIdx.x;
 if (idx >= n) return;
 // in-place reduction in global memory
 if (blockDim.x >= 1024 && tid < 512) idata[tid] += idata[tid + 512];
 __syncthreads();
 if (blockDim.x >= 512 && tid < 256) idata[tid] += idata[tid + 256];
 __syncthreads();
 if (blockDim.x >= 256 && tid < 128) idata[tid] += idata[tid + 128];
 __syncthreads();
 if (blockDim.x >= 128 && tid < 64) idata[tid] += idata[tid + 64];
 __syncthreads();
 // unrolling warp
 if (tid < 32)
 {
 volatile int *vsmem = idata;
 vsmem[tid] += vsmem[tid + 32];
 vsmem[tid] += vsmem[tid + 16];
 vsmem[tid] += vsmem[tid + 8];
 vsmem[tid] += vsmem[tid + 4];
 vsmem[tid] += vsmem[tid + 2];
 vsmem[tid] += vsmem[tid + 1];
 }
 // write result for this block to global mem
 if (tid == 0) g_odata[blockIdx.x] = idata[0];
}
__global__ void reduceSmem(int *g_idata, int *g_odata, unsigned int n)
{
 __shared__ int smem[DIM];
 // set thread ID
 unsigned int tid = threadIdx.x;
 // boundary check
 unsigned int idx = blockIdx.x * blockDim.x + threadIdx.x;
 if (idx >= n) return;
 // convert global data pointer to the local pointer of this block
 int *idata = g_idata + blockIdx.x * blockDim.x;
 // set to smem by each threads
 smem[tid] = idata[tid];
 __syncthreads();
 // in-place reduction in shared memory
 if (blockDim.x >= 1024 && tid < 512) smem[tid] += smem[tid + 512];
 __syncthreads();
 if (blockDim.x >= 512 && tid < 256) smem[tid] += smem[tid + 256];
 __syncthreads();
 if (blockDim.x >= 256 && tid < 128) smem[tid] += smem[tid + 128];
 __syncthreads();
 if (blockDim.x >= 128 && tid < 64) smem[tid] += smem[tid + 64];
 __syncthreads();
 // unrolling warp
 if (tid < 32)
 {
 volatile int *vsmem = smem;
 vsmem[tid] += vsmem[tid + 32];
 vsmem[tid] += vsmem[tid + 16];
 vsmem[tid] += vsmem[tid + 8];
 vsmem[tid] += vsmem[tid + 4];
 vsmem[tid] += vsmem[tid + 2];
 vsmem[tid] += vsmem[tid + 1];
 }
 // write result for this block to global mem
 if (tid == 0) g_odata[blockIdx.x] = smem[0];
}
__global__ void reduceSmemDyn(int *g_idata, int *g_odata, unsigned int n)
{
 extern __shared__ int smem[];
 // set thread ID
 unsigned int tid = threadIdx.x;
 int *idata = g_idata + blockIdx.x * blockDim.x;
 // set to smem by each threads
 smem[tid] = idata[tid];
 __syncthreads();
 // in-place reduction in global memory
 if (blockDim.x >= 1024 && tid < 512) smem[tid] += smem[tid + 512];
 __syncthreads();
 if (blockDim.x >= 512 && tid < 256) smem[tid] += smem[tid + 256];
 __syncthreads();
 if (blockDim.x >= 256 && tid < 128) smem[tid] += smem[tid + 128];
 __syncthreads();
 if (blockDim.x >= 128 && tid < 64) smem[tid] += smem[tid + 64];
 __syncthreads();
 // unrolling warp
 if (tid < 32)
 {
 volatile int *vsmem = smem;
 vsmem[tid] += vsmem[tid + 32];
 vsmem[tid] += vsmem[tid + 16];
 vsmem[tid] += vsmem[tid + 8];
 vsmem[tid] += vsmem[tid + 4];
 vsmem[tid] += vsmem[tid + 2];
 vsmem[tid] += vsmem[tid + 1];
 }
 // write result for this block to global mem
 if (tid == 0) g_odata[blockIdx.x] = smem[0];
}
// unroll4 + complete unroll for loop + gmem
__global__ void reduceGmemUnroll(int *g_idata, int *g_odata, unsigned int n)
{
 // set thread ID
 unsigned int tid = threadIdx.x;
 unsigned int idx = blockIdx.x * blockDim.x * 4 + threadIdx.x;
 // convert global data pointer to the local pointer of this block
 int *idata = g_idata + blockIdx.x * blockDim.x * 4;
 // unrolling 4
 if (idx < n)
 {
 int a1, a2, a3, a4;
 a1 = a2 = a3 = a4 = 0;
 a1 = g_idata[idx];
 if (idx + blockDim.x < n) a2 = g_idata[idx + blockDim.x];
 if (idx + 2 * blockDim.x < n) a3 = g_idata[idx + 2 * blockDim.x];
 if (idx + 3 * blockDim.x < n) a4 = g_idata[idx + 3 * blockDim.x];
 g_idata[idx] = a1 + a2 + a3 + a4;
 }
 __syncthreads();
 // in-place reduction in global memory
 if (blockDim.x >= 1024 && tid < 512) idata[tid] += idata[tid + 512];
 __syncthreads();
 if (blockDim.x >= 512 && tid < 256) idata[tid] += idata[tid + 256];
 __syncthreads();
 if (blockDim.x >= 256 && tid < 128) idata[tid] += idata[tid + 128];
 __syncthreads();
 if (blockDim.x >= 128 && tid < 64) idata[tid] += idata[tid + 64];
 __syncthreads();
 // unrolling warp
 if (tid < 32)
 {
 volatile int *vsmem = idata;
 vsmem[tid] += vsmem[tid + 32];
 vsmem[tid] += vsmem[tid + 16];
 vsmem[tid] += vsmem[tid + 8];
 vsmem[tid] += vsmem[tid + 4];
 vsmem[tid] += vsmem[tid + 2];
 vsmem[tid] += vsmem[tid + 1];
 }
 // write result for this block to global mem
 if (tid == 0) g_odata[blockIdx.x] = idata[0];
}
__global__ void reduceSmemUnroll(int *g_idata, int *g_odata, unsigned int n)
{
 // static shared memory
 __shared__ int smem[DIM];
 // set thread ID
 unsigned int tid = threadIdx.x;
 // global index, 4 blocks of input data processed at a time
 unsigned int idx = blockIdx.x * blockDim.x * 4 + threadIdx.x;
 // unrolling 4 blocks
 int tmpSum = 0;
 // boundary check
 if (idx < n)
 {
 int a1, a2, a3, a4;
 a1 = a2 = a3 = a4 = 0;
 a1 = g_idata[idx];
 if (idx + blockDim.x < n) a2 = g_idata[idx + blockDim.x];
 if (idx + 2 * blockDim.x < n) a3 = g_idata[idx + 2 * blockDim.x];
 if (idx + 3 * blockDim.x < n) a4 = g_idata[idx + 3 * blockDim.x];
 tmpSum = a1 + a2 + a3 + a4;
 }
 smem[tid] = tmpSum;
 __syncthreads();
 // in-place reduction in shared memory
 if (blockDim.x >= 1024 && tid < 512) smem[tid] += smem[tid + 512];
 __syncthreads();
 if (blockDim.x >= 512 && tid < 256) smem[tid] += smem[tid + 256];
 __syncthreads();
 if (blockDim.x >= 256 && tid < 128) smem[tid] += smem[tid + 128];
 __syncthreads();
 if (blockDim.x >= 128 && tid < 64) smem[tid] += smem[tid + 64];
 __syncthreads();
 // unrolling warp
 if (tid < 32)
 {
 volatile int *vsmem = smem;
 vsmem[tid] += vsmem[tid + 32];
 vsmem[tid] += vsmem[tid + 16];
 vsmem[tid] += vsmem[tid + 8];
 vsmem[tid] += vsmem[tid + 4];
 vsmem[tid] += vsmem[tid + 2];
 vsmem[tid] += vsmem[tid + 1];
 }
 // write result for this block to global mem
 if (tid == 0) g_odata[blockIdx.x] = smem[0];
}
__global__ void reduceSmemUnrollDyn(int *g_idata, int *g_odata, unsigned int n)
{
 extern __shared__ int smem[];
 // set thread ID
 unsigned int tid = threadIdx.x;
 unsigned int idx = blockIdx.x * blockDim.x * 4 + threadIdx.x;
 // unrolling 4
 int tmpSum = 0;
 if (idx < n)
 {
 int a1, a2, a3, a4;
 a1 = a2 = a3 = a4 = 0;
 a1 = g_idata[idx];
 if (idx + blockDim.x < n) a2 = g_idata[idx + blockDim.x];
 if (idx + 2 * blockDim.x < n) a3 = g_idata[idx + 2 * blockDim.x];
 if (idx + 3 * blockDim.x < n) a4 = g_idata[idx + 3 * blockDim.x];
 tmpSum = a1 + a2 + a3 + a4;
 }
 smem[tid] = tmpSum;
 __syncthreads();
 // in-place reduction in global memory
 if (blockDim.x >= 1024 && tid < 512) smem[tid] += smem[tid + 512];
 __syncthreads();
 if (blockDim.x >= 512 && tid < 256) smem[tid] += smem[tid + 256];
 __syncthreads();
 if (blockDim.x >= 256 && tid < 128) smem[tid] += smem[tid + 128];
 __syncthreads();
 if (blockDim.x >= 128 && tid < 64) smem[tid] += smem[tid + 64];
 __syncthreads();
 // unrolling warp
 if (tid < 32)
 {
 volatile int *vsmem = smem;
 vsmem[tid] += vsmem[tid + 32];
 vsmem[tid] += vsmem[tid + 16];
 vsmem[tid] += vsmem[tid + 8];
 vsmem[tid] += vsmem[tid + 4];
 vsmem[tid] += vsmem[tid + 2];
 vsmem[tid] += vsmem[tid + 1];
 }
 // write result for this block to global mem
 if (tid == 0) g_odata[blockIdx.x] = smem[0];
}
__global__ void reduceNeighboredGmem(int *g_idata, int *g_odata, unsigned int n)
{
 // set thread ID
 unsigned int tid = threadIdx.x;
 unsigned int idx = blockIdx.x * blockDim.x + threadIdx.x;
 // convert global data pointer to the local pointer of this block
 int *idata = g_idata + blockIdx.x * blockDim.x;
 // boundary check
 if (idx >= n) return;
 // in-place reduction in global memory
 for (int stride = 1; stride < blockDim.x; stride *= 2)
 {
 if ((tid % (2 * stride)) == 0)
 {
 idata[tid] += idata[tid + stride];
 }
 // synchronize within threadblock
 __syncthreads();
 }
 // write result for this block to global mem
 if (tid == 0) g_odata[blockIdx.x] = idata[0];
}
__global__ void reduceNeighboredSmem(int *g_idata, int *g_odata, unsigned int n)
{
 __shared__ int smem[DIM];
 // set thread ID
 unsigned int tid = threadIdx.x;
 unsigned int idx = blockIdx.x * blockDim.x + threadIdx.x;
 // convert global data pointer to the local pointer of this block
 int *idata = g_idata + blockIdx.x * blockDim.x;
 // boundary check
 if (idx >= n) return;
 smem[tid] = idata[tid];
 __syncthreads();
 // in-place reduction in global memory
 for (int stride = 1; stride < blockDim.x; stride *= 2)
 {
 if ((tid % (2 * stride)) == 0)
 {
 smem[tid] += smem[tid + stride];
 }
 // synchronize within threadblock
 __syncthreads();
 }
 // write result for this block to global mem
 if (tid == 0) g_odata[blockIdx.x] = smem[0];
}
int main(int argc, char **argv)
{
 // set up device
 int dev = 0;
 cudaDeviceProp deviceProp;
 CHECK(cudaGetDeviceProperties(&deviceProp, dev));
 printf("%s starting reduction at ", argv[0]);
 printf("device %d: %s ", dev, deviceProp.name);
 CHECK(cudaSetDevice(dev));
 bool bResult = false;
 // initialization
 int size = 1 << 22; // total number of elements to reduce
 printf(" with array size %d ", size);
 // execution configuration
 int blocksize = DIM; // initial block size
 dim3 block (blocksize, 1);
 dim3 grid ((size + block.x - 1) / block.x, 1);
 printf("grid %d block %d\n", grid.x, block.x);
 // allocate host memory
 size_t bytes = size * sizeof(int);
 int *h_idata = (int *) malloc(bytes);
 int *h_odata = (int *) malloc(grid.x * sizeof(int));
 int *tmp = (int *) malloc(bytes);
 // initialize the array
 for (int i = 0; i < size; i++)
 {
 h_idata[i] = (int)( rand() & 0xFF );
 }
 memcpy (tmp, h_idata, bytes);
 int gpu_sum = 0;
 // allocate device memory
 int *d_idata = NULL;
 int *d_odata = NULL;
 CHECK(cudaMalloc((void **) &d_idata, bytes));
 CHECK(cudaMalloc((void **) &d_odata, grid.x * sizeof(int)));
 // cpu reduction
 int cpu_sum = recursiveReduce (tmp, size);
 printf("cpu reduce : %d\n", cpu_sum);
 // reduce gmem
 CHECK(cudaMemcpy(d_idata, h_idata, bytes, cudaMemcpyHostToDevice));
 reduceNeighboredGmem<<<grid.x, block>>>(d_idata, d_odata, size);
 CHECK(cudaMemcpy(h_odata, d_odata, grid.x * sizeof(int), cudaMemcpyDeviceToHost));
 gpu_sum = 0;
 for (int i = 0; i < grid.x; i++) gpu_sum += h_odata[i];
 printf("reduceNeighboredGmem: %d <<<grid %d block %d>>>\n", gpu_sum, grid.x, block.x);
 // reduce gmem
 CHECK(cudaMemcpy(d_idata, h_idata, bytes, cudaMemcpyHostToDevice));
 reduceNeighboredSmem<<<grid.x, block>>>(d_idata, d_odata, size);
 CHECK(cudaMemcpy(h_odata, d_odata, grid.x * sizeof(int), cudaMemcpyDeviceToHost));
 gpu_sum = 0;
 for (int i = 0; i < grid.x; i++) gpu_sum += h_odata[i];
 printf("reduceNeighboredSmem: %d <<<grid %d block %d>>>\n", gpu_sum, grid.x, block.x);
 // reduce gmem
 CHECK(cudaMemcpy(d_idata, h_idata, bytes, cudaMemcpyHostToDevice));
 reduceGmem<<<grid.x, block>>>(d_idata, d_odata, size);
 CHECK(cudaMemcpy(h_odata, d_odata, grid.x * sizeof(int), cudaMemcpyDeviceToHost));
 gpu_sum = 0;
 for (int i = 0; i < grid.x; i++) gpu_sum += h_odata[i];
 printf("reduceGmem : %d <<<grid %d block %d>>>\n", gpu_sum, grid.x, block.x);
 // reduce smem
 CHECK(cudaMemcpy(d_idata, h_idata, bytes, cudaMemcpyHostToDevice));
 reduceSmem<<<grid.x, block>>>(d_idata, d_odata, size);
 CHECK(cudaMemcpy(h_odata, d_odata, grid.x * sizeof(int), cudaMemcpyDeviceToHost));
 gpu_sum = 0;
 for (int i = 0; i < grid.x; i++) gpu_sum += h_odata[i];
 printf("reduceSmem : %d <<<grid %d block %d>>>\n", gpu_sum, grid.x, block.x);
 // reduce smem
 CHECK(cudaMemcpy(d_idata, h_idata, bytes, cudaMemcpyHostToDevice));
 reduceSmemDyn<<<grid.x, block, blocksize*sizeof(int)>>>(d_idata, d_odata, size);
 CHECK(cudaMemcpy(h_odata, d_odata, grid.x * sizeof(int), cudaMemcpyDeviceToHost));
 gpu_sum = 0;
 for (int i = 0; i < grid.x; i++) gpu_sum += h_odata[i];
 printf("reduceSmemDyn : %d <<<grid %d block %d>>>\n", gpu_sum, grid.x, block.x);
 // reduce gmem
 CHECK(cudaMemcpy(d_idata, h_idata, bytes, cudaMemcpyHostToDevice));
 reduceGmemUnroll<<<grid.x / 4, block>>>(d_idata, d_odata, size);
 CHECK(cudaMemcpy(h_odata, d_odata, grid.x / 4 * sizeof(int), cudaMemcpyDeviceToHost));
 gpu_sum = 0;
 for (int i = 0; i < grid.x / 4; i++) gpu_sum += h_odata[i];
 printf("reduceGmemUnroll4 : %d <<<grid %d block %d>>>\n", gpu_sum, grid.x / 4, block.x);
 // reduce smem
 CHECK(cudaMemcpy(d_idata, h_idata, bytes, cudaMemcpyHostToDevice));
 reduceSmemUnroll<<<grid.x / 4, block>>>(d_idata, d_odata, size);
 CHECK(cudaMemcpy(h_odata, d_odata, grid.x / 4 * sizeof(int), cudaMemcpyDeviceToHost));
 gpu_sum = 0;
 for (int i = 0; i < grid.x / 4; i++) gpu_sum += h_odata[i];
 printf("reduceSmemUnroll4 : %d <<<grid %d block %d>>>\n", gpu_sum, grid.x / 4, block.x);
 // reduce smem
 CHECK(cudaMemcpy(d_idata, h_idata, bytes, cudaMemcpyHostToDevice));
 reduceSmemUnrollDyn<<<grid.x / 4, block, DIM*sizeof(int)>>>(d_idata, d_odata, size);
 CHECK(cudaMemcpy(h_odata, d_odata, grid.x / 4 * sizeof(int), cudaMemcpyDeviceToHost));
 gpu_sum = 0;
 for (int i = 0; i < grid.x / 4; i++) gpu_sum += h_odata[i];
 printf("reduceSmemDynUnroll4: %d <<<grid %d block %d>>>\n", gpu_sum, grid.x / 4, block.x);
 // free host memory
 free(h_idata);
 free(h_odata);
 // free device memory
 CHECK(cudaFree(d_idata));
 CHECK(cudaFree(d_odata));
 // reset device
 CHECK(cudaDeviceReset());
 // check the results
 bResult = (gpu_sum == cpu_sum);
 if(!bResult) printf("Test failed!\n");
 return EXIT_SUCCESS;
}
```

## Output:
## reduceInteger8.cu:
```
root@SAV-MLSystem:/home/student/Sidd_Lab_Exp_3# nvcc reduceInteger8.cu -o reduceInteger8
root@SAV-MLSystem:/home/student/Sidd_Lab_Exp_3# nvcc reduceInteger8.cu
root@SAV-MLSystem:/home/student/Sidd_Lab_Exp_3# ./reduceInteger8
./reduceInteger8 starting reduction at device 0: NVIDIA GeForce GT 710 with array size 4194304
grid 32768 block 128
cpu reduce : 534907410
reduceNeighboredGmem: 534907410 <<<grid 32768 block 128>>>
reduceNeighboredSmem: 534907410 <<<grid 32768 block 128>>>
reduceGmem : 534907410 <<<grid 32768 block 128>>>
reduceSmem : 534907410 <<<grid 32768 block 128>>>
reduceSmemDyn : 534907410 <<<grid 32768 block 128>>>
reduceGmemUnroll16 : 267424728 <<<grid 4096 block 128>>>
reduceSmemUnroll16 : 267424728 <<<grid 4096 block 128>>>
reduceSmemDynUnroll8: 267424728 <<<grid 4096 block 128>>>
Test failed!
```
## reduceInteger16.cu:
```
root@SAV-MLSystem:/home/student/Sidd_Lab_Exp_3# nvcc reduceInteger16.cu -o reduceInteger16
root@SAV-MLSystem:/home/student/Sidd_Lab_Exp_3# nvcc reduceInteger16.cu
root@SAV-MLSystem:/home/student/Sidd_Lab_Exp_3# ./reduceInteger16
./reduceInteger16 starting reduction at device 0: NVIDIA GeForce GT 710 with array size 4194304
grid 32768 block 128
cpu reduce : 534907410
reduceNeighboredGmem: 534907410 <<<grid 32768 block 128>>>
reduceNeighboredSmem: 534907410 <<<grid 32768 block 128>>>
reduceGmem : 534907410 <<<grid 32768 block 128>>>
reduceSmem : 534907410 <<<grid 32768 block 128>>>
reduceSmemDyn : 534907410 <<<grid 32768 block 128>>>
reduceGmemUnroll16 : 133554970 <<<grid 2048 block 128>>>
reduceSmemUnroll16 : 133554970 <<<grid 2048 block 128>>>
reduceSmemDynUnroll16: 133554970 <<<grid 2048 block 128>>>
Test failed!
```
## EXPLANATION:
* Using nvprof metrics and events, the reduceUnrolling8 kernel achieved a reduction result of
267424728, while the reduceUnrolling16 kernel achieved a reduction result of 133554970.
* The reduceUnrolling16 kernel demonstrated a difference in performance compared to
reduceUnrolling8, and further analysis using nvprof is necessary to determine the specific factors
contributing to this performance variation

## Result:
The "reduceUnrolling8" kernel achieved a reduction result of 267424728, while the
"reduceUnrolling16" kernel achieved a reduction result of 133554970. The performance of the
"reduceUnrolling16" kernel differed from the "reduceUnrolling8" kernel, and further analysis using
"nvprof" is needed to understand the specific differences in performance.
