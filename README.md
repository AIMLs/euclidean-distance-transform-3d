## Multi-Label 3D Euclidean Distance Transform (MLEDT-3D)

Compute the Euclidean Distance Transform of a 1d, 2d, or 3d labeled image containing multiple labels in a single pass with support for anisotropic dimensions.

### Python Installation

*Requires a C++ compiler*

```bash
cd python
pip install numpy
python setup.py develop
```

### Python Usage

Consult `help(edt)` after importing. The edt module contains: `edt`, `edtsq` which compute the euclidean and squared euclidean distance respectively and select dimension based on the shape of the numpy object fed to them. 1D, 2D, and 3D volumes are supported. 1D processing is extremely fast.  

If for some reason you'd like to use the specific 'D' function you want, `edt1d`, `edt1dsq`, `edt2d`, `edt2dsq`, `edt3d`, and `edt3dsq` are included.

```python
import edt
import numpy as np

labels = np.ones(shape=(512, 512, 512), dtype=np.uint32)
dt = edt.edt(labels, anisotropy=(6, 6, 30)) # e.g. for the S1 dataset by Kasthuri et al., 2014
```

### C++ Usage

Include the edt.hpp header/implementation. The function names are underscored to avoid a namespace collision in Cython.   

```cpp
#include "edt.hpp"

int main () {

	int* labels = new int[3*3*3]();

	int sx = 3, sy = 3, sz = 3;
	float wx = 6, wy = 6, wz = 30; // anisotropy

	float* dt = edt::_edt3d(labels, sx, sy, sz, wx, wy, wz)

	return 0;
}
```

For compilation, I recommend the compiler flags `-O3` and `-ffast-math`.  


### Motivation

The connectomics field commonly generates very large densely labeled volumes of neural tissue. Some algorithms, such as the TEASAR skeletonization algorithm [1] require the computation of a 3D Euclidean Distance Transform. We found that the commodity implementation of the distance transform as implemented in [scipy](https://github.com/scipy/scipy/blob/f3dd9cba8af8d3614c88561712c967a9c67c2b50/scipy/ndimage/src/ni_morphology.c) (implementing the Voronoi based method of Maurer et al. [2]) was too slow for our needs.  

The relatively speedy scipy implementation took about 20 seconds to compute the transform of a 512x512x512 binary image. Unfortunately, there are typically more than 300 distinct labels within a volume, requiring the serial application of the EDT. While cropping to the ROI does help, many ROIs are diagonally oriented and span the volume, requiring a full EDT. I found that in our numpy/scipy based implementation of TEASAR, EDT was taking approximately a quarter of the time on its own. The amount of time the algorithm was taking per a block was estimated to be multiple hours per a core. 

Also concerning for our TEASAR implementation was that the scipy version did not have a parameter for handling anisotropic data sets (though Maurer et al is clear that their algorithm supports anisotropy with a simple modification). 

I realized that it's possible to compute the EDT much more quickly by computing the distance transform in one pass by making it label boundary aware at slightly higher computational cost. Since the distance transform does not result in overlapping boundaries, it is trivial to then perform a fast masking operation to query the block for the appropriate distance transform. 

The implementation presented here uses concepts from the 1994 paper by T. Saito and Toriwaki [3] \(note: not the M. Saito of the TEASAR paper) and combines the 1966 linear sweeping method of Rosenfeld and Pfaltz [4] with that of Felzenszwald and Huttenlocher's 2012 two pass linear time parabolic minmal envelope method [5]. I incorporate a few minor modifications to the algorithms to remove the necessity of a black border. My own contribution here is the modification of both the linear sweep and parabolic methods to account for multiple label boundaries.

This implementation was able to compute the distance transform of a binary image in 7-8 seconds. When adding in the multiple boundary modification, this rose to 9 seconds. 


### References

1. M. Sato, et al. "TEASAR: Tree-structure Extraction Algorithm for Accurate and Robust Skeletons". Proc. 8th Pacific Conf. on Computer Graphics and Applications. Oct. 2000. doi: 10.1109/PCCGA.2000.883951
2. C. Maurer, et al. "A Linear Time Algorithm for Computing Exact Euclidean Distance Transforms of Binary Images in Arbitrary Dimensions". IEEE Transactions on Pattern Analysis and Machine Intelligence. Vol. 25, No. 2. February 2003. doi: 10.1109/TPAMI.2003.1177156
3. T. Saito and J. Toriwaki. "New Algorithms for Euclidean Distance Transformation of an n-Dimensional Digitized Picture with Applications". Pattern Recognition, Vol. 27, Issue 11, Nov. 1994, Pg. 1551-1565. doi: 10.1016/0031-3203(94)90133-3
4. A. Rosenfeld and J. Pfaltz. "Sequential Operations in Digital Picture Processing". Journal of the ACM. Vol. 13, Issue 4, Oct. 1966, Pg. 471-494. doi: 10.1145/321356.321357
5. P. Felzenszwald and D. Huttenlocher. "Distance Transforms of Sampled Functions". Theory of Computing, Vol. 8, 2012, Pg. 415-428. doi: 10.4086/toc.2012.v008a019
