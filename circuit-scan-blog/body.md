{{ "1" | chapter("Introduction") }}

{{ "1.1" | subchapter("Preface") }}

I dislike the fact that ltspice (circuit simulation program) that we use for school is so annoying and there are no ways to rebind shortcuts especially on mac, where the function keys require me to awkwardly contort my hands to use the dumb unintuitive default keybinds.
 
{{ "1.2" | subchapter("General Overview") }}

An abstracted overview of how the algorithm works is by first processing the image with a machine learning model (I used YOLOv11 because its easy) to seperate components from wiring and text. Then, through further processing it extracts a graph mapping each component to the componenets next to it based on the wiring. Then, for each component that is not directionally independent, I run it through a seperate model with rotation data to determine rotation (my trainings of that model were worse at classification). Furthermore, text is identified, processed using a custom trained character recognition model, and matched to each component based on standard naming practices, with some inferences. Finally, the output is converted into an LTSpice file where it can be opened and simulated.

{{ "2" | chapter("In Depth Process Analysis") }}

{{ "2.1" | subchapter("Trained Models Using YOLOv11") }}

Although YOLO is notorious for it's scummy business practices, it has an abstraction that is so simple that even I can use it without learning much.[1]

I converted the dataset to a YOLO OB format with a simple python script, and then trained it using these parameters (what I found best from my testing):

- initial learning rate = 0.00406 

- box = 0.02, dfl = 0.1 -> focus on classification vs having extremely accurate bounding boxes

- cls = 1.0 

Training mostly plataued in improvements at around 500-600 epochs, with it overfitting past that in my experience. 

{{ "2.1" | subchapter("Constructing the Graph") }}

After I found a model that was accurate enough for the continuation of progress, I worked on creating an algorithm that converts this:
{{ metadata.images.ml_circuit_diagram | image_process("", "large") }}

into something that I could process algorithmically:

{{ metadata.images.list_circuit_diagram | image_process("", "large") }}

For context, OpenCV processes images as a numpy 2d array of pixel color values. Essentially, by converting this into a binary array of 0 or 1 values, where 0 represents some path, and 255 (white) representing a untravelable area, it essentially becomes a simple graphing problem. Therefor, for each node I converted it into a black box ensuring continuity on a single component, and then just perform DFS where I append unvisited pixels to the stack and explore from there. 
{{ metadata.images.bounding_circuit_diagram | image_process("", "large") }}
To determine if the pixel is inside another bounding box, which would essentially mean that one component is connected to the other, instead of comparing the pixel with every single box, I created a KDtree (think binary search but for multiple dimensions) of center points of the bounding boxes, and only checked each pixel against each bounding box. Finally, if the pixel in BFS was in another bounding box, it'd not continue and add a connection in the adjacency list of one node to the other.

{{ "2.2" | subchapter("Matching Labels To Components") }}

{{ "2.3" | subchapter("Complex Manhattan Grid Algorithm") }}

{{ "2.4" | subchapter("Converting To LTSpice") }}

I couldn't find documentation for how LTspice formats its .asc files, so I just did some experimentation myself. 

Wires are in the format [WIRE 0 158 0 273, WIRE 0 158 0 158] where it represents the start and the end of the wire. To handle the parsing for wires, by assuming wires are always directly parallel to another node, it only draws wires up/down in the direction of the connected nodes. 

Components also are weird in the sense that their coordinates are not determined by the center but one of the corners, and that means that each direction of rotation will have its own offset. Therefor, I had to manually apply a transform for each component's directions.

To ensure small imperfections in the code don't cause disconnectivity, I also have a function to snap all components to a set grid size of 16 pixels, rounding to +- 16 pixels.

{{ "3" | chapter("Demo") }}


{{ "4" | chapter("References") }}

Bayer, J. (n.d.). CGHD1152 [Dataset]. Kaggle. Retrieved from https://www.kaggle.com/datasets/johannesbayer/cghd1152