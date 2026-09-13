README – Unreal Engine Wildfire Simulation and Drone Data Capture
This project is an Unreal Engine 5 wildfire simulation designed to generate synthetic wildfire data for machine-learning and computer-vision research. The simulation models wildfire spread across a grassland environment using a simplified Rothermel-based rate-of-spread model, dynamic weather conditions, vegetation burning, fire and smoke visual effects, and a controllable aerial drone.
The goal of the project is to create repeatable wildfire scenarios in a 3D environment so that aerial imagery and associated simulation information can later be collected for training and evaluating wildfire detection and forecasting models.

1. The Project Uses
•	Unreal Engine 5 for the wildfire simulation environment
•	Blueprints for wildfire behavior, weather, drone controls, and simulation logic
•	Procedural Content Generation (PCG) for vegetation placement
•	Instanced Static Mesh Components for efficient rendering of large amounts of grass
•	Niagara for fire and smoke visual effects
•	Rothermel fire-spread equations for estimating wildfire Rate of Spread (ROS)
•	Enhanced Input for drone controls
•	Scene Capture Components for future aerial image and sensor-data collection
•	GaeaUnrealTools and GaeaToolsEditor Plugins

2. Main Simulation Systems
The project is divided into several major Blueprint systems.
BP_FireManager
BP_FireManager controls the overall wildfire simulation.
Its responsibilities include:
•	Initializing the fire simulation
•	Creating the initial ignition point
•	Tracking active fire patches
•	Preventing previously burned cells from reigniting
•	Determining neighboring cells that may ignite
•	Calling the Rate of Spread calculation (from RothermelROS)
•	Managing grass burning
•	Maintaining the grass lookup structure
•	Coordinating fire spread over time
Important variables include:
•	ActiveFirePatches
•	UsedCellIndices
•	GrassLookup
•	GridSize
•	CellSize
•	Wind and fuel-moisture information obtained from the weather system

BP_FirePatch
BP_FirePatch represents an individual burning region of the wildfire.
Each fire patch:
•	Is associated with a grid cell
•	Contains fire and smoke visual effects
•	Tracks its burn duration
•	Attempts to spread fire into neighboring cells
•	Gradually transitions through its burning lifecycle
•	Is removed after its burn duration has completed
A fire cell that has already burned is stored in UsedCellIndices, preventing the simulation from repeatedly igniting the same location.

BP_WeatherSystem
BP_WeatherSystem controls environmental conditions that influence wildfire spread.
Current weather variables include:
•	Wind Speed
•	Wind Direction
•	Dead Fuel Moisture
The weather system also contains a wind-gust system.
Wind gusts:
1.	Occur after a random interval.
2.	Increase the normal wind speed by a random amount.
3.	Gradually return to the normal wind speed.
4.	Schedule the next gust after the previous gust finishes.
This allows fire-spread conditions to change during a simulation instead of remaining completely constant.

3. Rothermel Rate-of-Spread Model
The wildfire spread calculation is implemented in:
BP_RothermelROS
The main function is:
CalcROS()
The simulation uses a simplified Rothermel surface-fire Rate of Spread model.
Inputs currently include:
•	WindSpeed
•	DeadFuelMoisture
•	Slope
The current implementation primarily uses wind speed and dead-fuel moisture when calculating ROS.
The model calculates several intermediate values, including:
•	Moisture Ratio
•	Moisture Damping
•	Reaction Intensity (IR)
•	Wind velocity in ft/min
•	Wind coefficient (PhiW)
•	Heat of pre-ignition (Qig)
•	No-wind Rate of Spread (R0)
•	Final Rate of Spread
The moisture ratio is calculated approximately as:
MoistureRatio = DeadFuelMoisture / 0.15
The moisture damping calculation uses:
MoistureDamping = 1 - 2.59M + 5.11M² - 3.52M³
where M is the moisture ratio.
If dead-fuel moisture reaches approximately 0.15 (can be changed depending on other factors), the current implementation returns a Rate of Spread of zero, meaning fire will not spread if the threshold is not met.
The calculated Rate of Spread is ultimately converted into approximately:
meters per minute
The implementation should therefore be described as a:
simplified Rothermel-based surface-fire spread model parameterized for the simulated grassland environment.
Primary reference:
Rothermel, R. C. (1972). A Mathematical Model for Predicting Fire Spread in Wildland Fuels. USDA Forest Service Research Paper INT-115.
https://research.fs.usda.gov/treesearch/32533 

4. Vegetation and Grass Burning
The environment contains large quantities of procedural grass.
To avoid repeatedly searching every grass instance whenever fire spreads, the simulation creates a lookup table when the simulation begins.
Two structures are used:
ST_GrassInstanceRef
Stores:
•	GrassComponent
•	InstanceIndex
ST_GrassCellData
Stores:
•	GrassInstances
This is an array of ST_GrassInstanceRef structures.
The Fire Manager stores these values in:
GrassLookup
which maps:
CellIndex → Grass instances located inside that cell
This allows the simulation to quickly locate vegetation associated with a burning grid cell.

5. Burning Grass Material
Grass instances use Per Instance Custom Data to visually represent burning vegetation.
The grass material reads:
PerInstanceCustomData(0)
The Fire Manager calls:
SetCustomDataValue
with approximately:
•	Custom Data Index = 0
•	Custom Data Value = 1
This allows individual grass instances to change appearance when their cell burns.
The grass can therefore transition from normal vegetation toward a burned or charred appearance without creating an individual Actor for every piece of grass.

6. BurnGrassAtCell()
The function:
BurnGrassAtCell(CellIndex)
performs the vegetation-burning process.
The basic process is:
1.	Receive the wildfire CellIndex.
2.	Search GrassLookup.
3.	Retrieve ST_GrassCellData.
4.	Loop through the grass instances stored for that cell.
5.	Retrieve the grass component and instance index.
6.	Change that instance's custom material data.
This system avoids performing expensive spatial searches every time a new fire patch appears.

7. Wildfire Spread
The environment is divided conceptually into wildfire cells.
Each cell can transition through states such as:
•	Unburned
•	Burning
•	Burned
When a fire patch attempts to spread, the Fire Manager determines which neighboring grid cells are valid.
Before creating another fire patch, the system checks:
UsedCellIndices
If the index is already present, the cell cannot ignite again.
If the cell has not previously burned:
1.	A new fire patch is created.
2.	The Cell Index is added to UsedCellIndices.
3.	The Actor is added to ActiveFirePatches.
4.	The fire patch begins burning.
5.	Vegetation associated with the cell is changed to its burned state.
This prevents fire from repeatedly respawning at locations that have already burned.

8. Fire and Smoke Effects
Fire visualization uses Niagara-based visual effects.
Each fire patch contains visual effects representing:
•	Flames
•	Smoke
•	Ember
•	Distortion
Fire appearance changes during the patch's lifetime.
The current system also reduces the visual scale of the fire as the patch approaches the end of its burn duration.
After the burn lifecycle finishes:
1.	The fire patch is removed from ActiveFirePatches.
2.	The Actor is destroyed.
The burned vegetation remains permanently visually modified after the fire disappears.

9. Drone System
The project contains:
BP_Drone
The drone is implemented as a Pawn.
Current components include:
•	DroneMesh
•	SpringArm
•	Camera
•	SceneCaptureComponent2D
•	FloatingPawnMovement
The drone uses Unreal Engine's Enhanced Input System.

10. Drone Controls
Drone input is defined using:
•	IA_DroneHorizontal
•	IA_DroneVertical
•	IMC_Drone
Current controls are:
•	W – Move Forward
•	S – Move Backward
•	A – Move Left
•	D – Move Right
•	E – Move Up
•	Q – Move Down
IA_DroneHorizontal uses an Axis2D input.
The horizontal input is separated into:
•	Y → Forward / Backward
•	X → Right / Left
Movement is applied using:
Add Movement Input
with:
•	GetActorForwardVector
•	GetActorRightVector
•	GetActorUpVector
The placed drone should use:
Auto Possess Player = Player 0
when manually controlling the drone.

11. Planned Aerial Data Collection
The drone is intended to act as an aerial data-collection platform.
Planned sensor outputs include:
•	RGB imagery
•	Infrared / thermal imagery
•	Potential segmentation masks
•	Drone position
•	Drone rotation
•	Simulation timestamp
•	Wind speed
•	Wind direction
•	Fuel moisture
•	Fire-spread information
These outputs can later be associated with each captured frame to create a structured synthetic wildfire dataset.

12. Infrared / Thermal Camera
The planned infrared system will use another:
SceneCaptureComponent2D
attached to BP_Drone.
A typical setup will be:
BP_Drone
•	Camera
•	RGB SceneCaptureComponent2D
•	Thermal SceneCaptureComponent2D
The Thermal Scene Capture will output to a dedicated Render Target.
A thermal post-process material can then convert simulation information into an infrared-style image.
The initial implementation may classify regions approximately as:
•	Active flames → very hot
•	Recently burned vegetation → warm
•	Normal vegetation → cooler
•	Terrain/background → cooler
A later version could associate simulated temperatures with fire and terrain materials to create more physically meaningful thermal imagery.

13. Typical Blueprint Layout
A simplified project layout may look like:
•	BP_FireManager – Controls wildfire initialization, spread, vegetation lookup, and active fire patches
•	BP_FirePatch – Represents individual burning wildfire cells
•	BP_RothermelROS – Calculates Rate of Spread
•	BP_WeatherSystem – Controls wind, wind direction, moisture, and wind gusts
•	BP_Drone – Controllable aerial observation platform
•	ST_GrassInstanceRef – Stores a grass component and instance index
•	ST_GrassCellData – Stores grass instances associated with a wildfire cell
•	IMC_Drone – Drone Enhanced Input Mapping Context
•	IA_DroneHorizontal – Horizontal drone movement input
•	IA_DroneVertical – Vertical drone movement input
•	Fire / Smoke Niagara Systems – Wildfire visual effects
•	Grass Materials – Vegetation materials supporting Per Instance Custom Data

14. How to Run the Simulation
Note: this is only necessary if you are setting up a new scene in which a new landscape is to be tested. Otherwise, set up is already done for the Marion landscape.
1.	Open the wildfire level in Unreal Engine.
2.	Ensure the PCG vegetation has generated correctly.
3.	Place BP_FireManager in the level.
4.	Place BP_WeatherSystem in the level.
5.	Place BP_Drone above the simulation area.
6.	Set the drone's:
Auto Possess Player = Player 0
7.	Configure simulation values such as:
•	Wind Speed
•	Wind Direction
•	Dead Fuel Moisture
•	Simulation Speed
8.	Press Play.
9.	The Fire Manager initializes the vegetation lookup.
10.	The initial fire patch is created.
11.	Fire begins spreading according to the simulation's ROS and neighbor-spread logic.
12.	Use WASD and Q/E to move the drone around the wildfire.

15. Important Simulation Parameters
Important parameters that may be adjusted during experiments include:
Wind Speed
Controls the influence of wind on wildfire Rate of Spread.
Higher wind speeds generally increase forward fire propagation in the current model.
Wind starts randomly between 5-15 mph and gusts randomly between 5-10 mph randomly.
Wind Direction
Controls the preferred direction of wildfire spread.
Dead Fuel Moisture
Controls how easily vegetation burns.
Higher fuel moisture decreases the calculated Rate of Spread.
Variable fuel moisture/density is yet to be implemented.
Simulation Speed
Controls the temporal speed of the wildfire simulation.
Grid Size
Controls the dimensions of the wildfire simulation grid.
Cell Size
Controls the physical spacing represented by each wildfire cell.
Burn Duration
Controls how long an individual fire patch remains active.

16. Current Limitations
The simulation is still under development.
Current limitations include:
•	The Rothermel implementation is simplified and uses fixed constants for several fuel-bed characteristics.
•	Slope is not yet fully incorporated into the final Rate of Spread calculation. This will need to be implemented for forest landscapes but for flat landscapes, it should not have much effect.
•	Fire spreads using discrete simulation cells rather than a fully continuous fire front.
•	Fire and smoke Niagara effects are primarily visual approximations.
•	Thermal imaging is not yet a physically calibrated infrared sensor simulation (next step).
•	Fuel characteristics are currently simplified for the grassland environment.
These simplifications allow the simulator to remain computationally practical while still producing variable and directional wildfire behavior.

17. Future Development
Planned improvements include:
•	Infrared / thermal drone imagery
•	RGB aerial imagery
•	Automatic Render Target capture
•	Automatic dataset export
•	Segmentation or fire masks
•	Camera position and orientation metadata
•	Weather metadata for every captured frame
•	Repeatable drone flight paths
•	Spline-based automated drone movement
•	Improved Rothermel fuel parameters
•	Slope-dependent fire spread
•	Additional vegetation/fuel types
•	Greater randomness in fire appearance and propagation
•	More realistic smoke behavior
•	Fire intensity and temperature modeling

18. Project Goal
The long-term goal is to use Unreal Engine to produce a large synthetic aerial wildfire dataset.
Unlike simple image augmentation, the simulation generates wildfire scenes through an evolving environment where:
•	Fire spreads over time.
•	Wind influences propagation.
•	Fuel moisture influences Rate of Spread.
•	Vegetation changes after burning.
•	Smoke and flames evolve dynamically.
•	A virtual drone can observe the wildfire from different locations and altitudes.
This provides both the rendered imagery and the underlying simulation information, allowing future machine-learning models to use controlled synthetic wildfire data for tasks such as wildfire detection, segmentation, tracking, and forecasting.
