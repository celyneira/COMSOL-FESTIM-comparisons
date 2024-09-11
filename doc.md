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
   - whether we use curve loops or wires depends on whether we use the `.occ` or `.geo` geometry kernels. `.occ` allows for direct construction of more complex features such as cylinders, whereas using `.geo` requires explicit user definition of all the points, surfaces and volumes that would make up the cylinder. 
-  Surfaces
-  Surface Loops
-  Volumes

Therefore, we will begin by defining the points of our square of tungsten.
```
p1 = gmsh.model.occ.addPoint(-15e-3, 15e-3, 0, lc)
p2 = gmsh.model.occ.addPoint(-15e-3, -15e-3, 0, lc)
p3 = gmsh.model.occ.addPoint(15e-3, 15e-3, 0, lc)
p4 = gmsh.model.occ.addPoint(15e-3, -15e-3, 0, lc)
```
These points are then joined together using lines, paying close attention to the direction that these lines are going.
```
line_1_2 = gmsh.model.occ.addLine(p1, p2)
line_3_1 = gmsh.model.occ.addLine(p3, p1)
line_2_4 = gmsh.model.occ.addLine(p2, p4)
line_4_3 = gmsh.model.occ.addLine(p4, p3)
```
Lines are then used to create curve loops or wires. The lines must be listed such that the loop follows the correct line direction, and forms a closed loop.
```
base_loop = gmsh.model.occ.addWire([line_1_2, line_2_4, line_4_3, line_3_1])
```

We can then define circles for the CuCrZr tube.

```
inner_circle = gmsh.model.occ.addCircle(0,0,0,5e-3)
outer_circle = gmsh.model.occ.addCircle(0,0,0,10e-3)

inner_circle_loop = gmsh.model.occ.addWire([inner_circle])
outer_circle_loop = gmsh.model.occ.addWire([outer_circle])
```

Surfaces are defined using these loops, where the first item in the list denotes the borders of the surface, and any others define holes within the surface.
```
base_surface = gmsh.model.occ.addPlaneSurface([base_loop, outer_circle_loop])
cylinder_surface = gmsh.model.occ.addPlaneSurface([outer_circle_loop, inner_circle_loop])
```
While we could then define another surface above the first and join them together, it is often easier to just perform an extrusion of each surface.

```
outer_layer_extrusion = gmsh.model.occ.extrude([(2, base_surface)], 0, 0, 5e-3, numElements=[100])
interface_layer_extrusion = gmsh.model.occ.extrude([(2, cylinder_surface)], 0, 0, 5e-3, numElements=[100])
```
Upon performing the extrusion, GMSH will define any necessary surfaces and volumes for us. However, this means that the surface of the outer cylinder will have been defined twice. Therefore is is necessary to remove any duplicate definitions via 
```
remove_overlap = gmsh.model.occ.remove_all_duplicates()
```



