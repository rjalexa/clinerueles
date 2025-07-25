Frontend will be a container served React/Typescript/Tailwind project that will be used to interface to the current api container backend endpoints in order to give a real time overview of the system, the queues and tasks.

The first tab is called dashboard and will be similar to the  image contained in the frontend/images/dashboard.png image but if possible the "Task Flow Overview" section will be displayed as a node-edge network using the Xyflow (React Flow v12+) library with a topology respecting the frontend/images/statusflow.png image but rearranged to be in landscape mode.

Only use the existing api container endpoints if possible and never add new ones to the backend if not after excplicit approval.

The dashboard will use the SSE equipment we have prepared to update the relevant frontend elements in realtime.

The other two pages we can select as tabs (Tasks history and Tasks Cleanup) will be empty placeholder pages for now.

All of this will be developed over the existing frontend/ directory