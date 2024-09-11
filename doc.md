##Using the GMSH Python_API to generate meshes

The Python API of GMSH can be used to create meshes that are useable in FESTIM. Here we will walk through its usage when creating a monoblock section consisting of tungsten surrounding a tube of CuCrZr, as such:
FULL MESH HERE!

Firstly, GMSH must be imported and initialised, before creating our model.

```
import gmsh as gmsh

gmsh.initialize()
gmsh.model.add("mesh")
```

We can set the size of our mesh using:
```
lc = 5e-4
gmsh.option.setNumber("Mesh.CharacteristicLengthFactor", lc)
```
Models in GMSH consist of a series of:
- Points
- Lines
-  Wires / Curve Loops
   - whether we use curve loops or wires depends on whether we use the `.occ` or `.geo` geometry kernels. '.occ' allows for direct construction of more complex features such as cylinders, whereas using `.geo` requires explicit user definition of all the points, surfaces and volumes that would make up the cylinder. 
-  Surfaces
-  Surface Loops
-  Volumes
-  
