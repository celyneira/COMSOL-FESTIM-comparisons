##Using the GMSH Python_API to generate meshes

The Python API of GMSH can be used to create meshes that are useable in FESTIM. Here we will walk through its usage when creating a monoblock subsection consisting of tungsten surrounding a tube of CuCrZr

<img width="948" alt="Screenshot 2024-09-11 at 17 09 10" src="https://github.com/user-attachments/assets/1ec52a67-840d-4ef8-a741-a79f0e2c1cc3">

Firstly, GMSH must be imported and initialised.

```
import gmsh as gmsh

gmsh.initialize()
gmsh.model.add("mesh")
```

We can set the size of our mesh using:
```
lc = 1e-3
```
Models in GMSH consist of a series of:
- Points
- Lines
-  Wires / Curve Loops
   - whether we use curve loops or wires depends on whether we use the `.occ` or `.geo` geometry kernels. `.occ` allows for direct construction of more complex features such as cylinders, whereas using `.geo` requires explicit user definition of all the points, surfaces and volumes that would make up the cylinder. 
-  Surfaces
-  Surface Loops
-  Volumes

We will begin by defining the points of our square of tungsten.
```
p1 = gmsh.model.occ.addPoint(-15e-3, 15e-3, 0, lc)
p2 = gmsh.model.occ.addPoint(-15e-3, -15e-3, 0, lc)
p3 = gmsh.model.occ.addPoint(15e-3, 15e-3, 0, lc)
p4 = gmsh.model.occ.addPoint(15e-3, -15e-3, 0, lc)
```
These points can then be joined together using lines. It is important that we pay close attention to the direction that these lines are going.
```
line_1_2 = gmsh.model.occ.addLine(p1, p2)
line_1_3 = gmsh.model.occ.addLine(p1, p3)
line_2_4 = gmsh.model.occ.addLine(p2, p4)
line_3_4 = gmsh.model.occ.addLine(p3, p4)
```
These are then used to create curve loops or wires. 
Wires and curve loops must be closed loops, and the list of lines must flow in the correct direction so as to form a complete loop.
```
base_loop = gmsh.model.occ.addWire([line_1_2, line_2_4, -line_3_4, -line_1_3])
```
We can also define the inner and outer circles and loops for the CuCrZr tube.

```
inner_circle = gmsh.model.occ.addCircle(0,0,0,5e-3)
outer_circle = gmsh.model.occ.addCircle(0,0,0,10e-3)

inner_circle_loop = gmsh.model.occ.addWire([inner_circle])
outer_circle_loop = gmsh.model.occ.addWire([outer_circle])
```
Surfaces are defined using loops, where the first loop in the list denotes the outer borders of the surface, and any others define holes within the surface. 
Here `base_surface` is our tungsten layer, and so it consists of our base rectangle curve loop, with a hole defined by the outer CuCrZr loop.
```
base_surface = gmsh.model.occ.addPlaneSurface([base_loop, outer_circle_loop])
cylinder_surface = gmsh.model.occ.addPlaneSurface([outer_circle_loop, inner_circle_loop])
```
While we could then define another surface above the first and join them together, it is often easier to just perform an extrusion of the surfaces. 
Here we stretch both the tungsten and CuCrZr surfaces by 5e-3 in the z-direction, and 0 in the x and y.

```
outer_layer_extrusion = gmsh.model.occ.extrude([(2, base_surface)], 0, 0, 5e-3, numElements=[100])
interface_layer_extrusion = gmsh.model.occ.extrude([(2, cylinder_surface)], 0, 0, 5e-3, numElements=[100])
```
Upon performing the extrusion, GMSH will define any necessary surfaces and volumes for us. However, this means that the surface of the outer cylinder will have been defined twice. Therefore it is necessary to remove any duplicate elements via 
```
remove_overlap = gmsh.model.occ.remove_all_duplicates()
```
It is important that all points in our model are defined using the same characteristic length. Therefore we need to define a couple of points across the mesh to have the same `lc`. Here we have used points on the inner and outer tube perimeters, on both the front and back of the mesh:

```
inner_front_perimiter_point = gmsh.model.occ.addPoint(5e-3, 0, 5e-3, lc)
inner_back_perimiter_point = gmsh.model.occ.addPoint(5e-3, 0, 0, lc)

outer_front_perimiter_point = gmsh.model.occ.addPoint(10e-3, 0, 5e-3, lc)
outer_back_perimiter_point = gmsh.model.occ.addPoint(10e-3, 0, 0, lc)

```
The model can then be synchronized:
```
gmsh.model.occ.synchronize()
```
At any point, the GMSH GUI can be opened by running the line
```
gmsh.fltk.run()
```
after synchronizing the model.
Running this command at this stage will open the GUI, displaying something that looks like this:

<img width="1710" alt="Screenshot 2024-09-11 at 14 36 44" src="https://github.com/user-attachments/assets/c292e6f2-7fb7-496e-896e-46acc3c21922">

To be used with FESTIM, it is necessary for us to define surface and volume markers. 

If the element has been defined explicitly, this is as easy as doing the following:
```
id_number = 1
gmsh.model.addPhysicalGroup(2, [base_surface, cylinder_surface], id_number, name = "surface")
```
where the 2 indicates that this is a 2nd dimension element, and we have listed the surfaces that we would like to assign with this ID number.

However, as we generated the surfaces using an extrusion, it can be complicated to keep track of which element corresponds to what.
GMSH assigns the surface labels cyclically when performing the extrusion, so these element IDs could be directly extracted using code. However, it may be more straightforward and intuitive to open the GUI as before and analyze the surfaces manually. 

After opening the GUI, again after synchronising and using `gmsh.fltk.run()`, go into 'Tools' then 'Options', and ensure that 'Surfaces' is checked under 'Geometry'.
This will make the surfaces are visible and selectable in the visualisation.

<img width="1710" alt="Screenshot 2024-09-11 at 14 37 53" src="https://github.com/user-attachments/assets/1700db41-2baf-4497-8941-b316190b4963">

We can then hover our mouse over each surface to see its information. For example, we can see that the front tungsten surface is defined as Plane 7, and borders the volume 1. 
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

The model must then be resynchronized before generating the mesh.
```
gmsh.model.occ.synchronize()

gmsh.model.mesh.generate(3)
```
The mesh can then be written to a file, and GMSH finalised. 
```
gmsh.write("my_mesh.msh")
gmsh.finalize()
```
We have now created our mesh! 

However, for use in FESTIM, our mesh now has to be converted into XDMF files, and the surfaces and volume IDs extracted.

This can be done using meshio via the following process:
```
import meshio
import numpy as np

msh = meshio.read("my_mesh.msh")

 # Initialize lists to store cells and their corresponding data
 triangle_cells_list = []
 tetra_cells_list = []
 triangle_data_list = []
 tetra_data_list = []

 # Extract cell data for all types
 for cell in msh.cells:
     if cell.type == "triangle":
         triangle_cells_list.append(cell.data)
     elif cell.type == "tetra":
         tetra_cells_list.append(cell.data)

 # Extract physical tags
 for key, data in msh.cell_data_dict["gmsh:physical"].items():
     if key == "triangle":
         triangle_data_list.append(data)
     elif key == "tetra":
         tetra_data_list.append(data)

 # Concatenate all tetrahedral cells and their data
 tetra_cells = np.concatenate(tetra_cells_list)
 tetra_data = np.concatenate(tetra_data_list)

 # Concatenate all triangular cells and their data
 triangle_cells = np.concatenate(triangle_cells_list)
 triangle_data = np.concatenate(triangle_data_list)

 # Create the tetrahedral mesh
 tetra_mesh = meshio.Mesh(
     points=msh.points,
     cells=[("tetra", tetra_cells)],
     cell_data={"f": [tetra_data]},
 )

 # Create the triangular mesh for the surface
 triangle_mesh = meshio.Mesh(
     points=msh.points,
     cells=[("triangle", triangle_cells)],
     cell_data={"f": [triangle_data]},
 )

# Write the mesh files
 meshio.write("volume_mesh.xdmf", tetra_mesh)
 meshio.write("surface_mesh.xdmf", triangle_mesh)
```
A FESTIM simulation can then be run:

```
import festim as F

model = F.Simulation()

model.mesh = F.MeshFromXDMF(volume_file ="volume_mesh.xdmf", boundary_file = "surface_mesh.xdmf")

model.materials = [F.Material(id=1, D_0=1, E_D=0),
                   F.Material(id=2, D_0=5, E_D=0)]

model.T = F.Temperature(800)

model.boundary_conditions = [F.DirichletBC(surfaces = [top_id], value = 1, field = 0),
                             F.DirichletBC(surfaces = [inner_cylinder_surface_id], value = 0, field = 0)]

model.exports = [F.XDMFExport("solute")]

model.settings = F.Settings(
    absolute_tolerance=1e-10,
    relative_tolerance=1e-10,
    transient=False,
)

model.initialise()
model.run()

```
This produces the following visualisation in Paraview:

<img width="910" alt="Screenshot 2024-09-11 at 17 57 57" src="https://github.com/user-attachments/assets/e310033a-8837-4d3f-adbd-fcf95e033cbf">






