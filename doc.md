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
Upon performing the extrusion, GMSH will define any necessary surfaces and volumes for us. However, this means that the surface of the outer cylinder will have been defined twice. Therefore it is necessary to remove any duplicate elements via 
```
remove_overlap = gmsh.model.occ.remove_all_duplicates()
```
The model must then be synchronized:
```
gmsh.model.occ.synchronize()
```
At any point, the GMSH GUI can be opened by running the line
```
gmsh.fltk.run()
```
after synchronizing the model.
Running this command at this stage will open the GUI, and you will see something that looks like this:

<img width="1710" alt="Screenshot 2024-09-11 at 14 36 44" src="https://github.com/user-attachments/assets/c292e6f2-7fb7-496e-896e-46acc3c21922">


To be used with FESTIM, it is necessary for us to define our surfaces and volumes. If you know the number of the surface / what the element is stored as this can be done using 
```
id_number = 1
gmsh.model.addPhysicalGroup(2, [base_surface, cylinder_surface], id_number, name = "surface")
```
where we list the surfaces that we would like to assign our required ID. 
As we generated the surfaces using an extrusion, while we could write code to extract the integers denoting each surface and volume, an alternative method that avoids the need to know what order they are generated and what corresponds to what is to open the GUI as before.

After doing this, by going into 'Tools' then 'Options', we can ensure that 'Surfaces' is checked under 'Geometry' such that the surfaces are visible in the GUI.

<img width="1710" alt="Screenshot 2024-09-11 at 14 37 53" src="https://github.com/user-attachments/assets/1700db41-2baf-4497-8941-b316190b4963">

We can then hover our mouse over the surface to see the corresponding surface information. For example, we can see that the front tungsten surface is defined as Plane 7, and borders the volume 1. 
<img width="418" alt="Screenshot 2024-09-11 at 14 38 29" src="https://github.com/user-attachments/assets/79841964-ecd4-4a57-b3f5-4f060f7850ec">

We can now look at each surface, and assign the necessary IDs.

```
front_id = 1
back_id = 2
left_id = 3
right_id = 4
top_id = 5
bottom_id = 6
outer_cylinder_surface_id = 7
inner_cylinder_surface_id = 8

tungsten_id = 1
cucrzr_id = 2

gmsh.model.addPhysicalGroup(2, [7, 10], front_id, name = "front")
gmsh.model.addPhysicalGroup(2, [6, 9], back_id, name = "back")
gmsh.model.addPhysicalGroup(2, [1], left_id, name = "left")
gmsh.model.addPhysicalGroup(2, [3], right_id, name = "right")
gmsh.model.addPhysicalGroup(2, [4], top_id, name = "top")
gmsh.model.addPhysicalGroup(2, [2], bottom_id, name = "bottom")
gmsh.model.addPhysicalGroup(2, [5], outer_cylinder_surface_id, name = "outer_cylinder")
gmsh.model.addPhysicalGroup(2, [8], inner_cylinder_surface_id, name = "inner_cylinder")

gmsh.model.addPhysicalGroup(3,[1], tungsten_id, name = "tungsten")
gmsh.model.addPhysicalGroup(3, [2], cucrzr_id, name = "cucrzr")
```

The model must then be resynchronized before we can generate the mesh, and write it to a file, before finalizing GMSH.
```
gmsh.model.occ.synchronize()

gmsh.fltk.run()
gmsh.model.mesh.generate(3)

gmsh.write("full_3d_mesh.msh")
gmsh.finalize()
```


