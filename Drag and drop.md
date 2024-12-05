# Drag and Drop Concept in React:

## Example Code:

```
import React, { useState } from "react";
import { DragDropContext, Draggable, Droppable } from "react-beautiful-dnd";
import { Box, Typography } from "@mui/material";

// Update initial modules to 30 items
const initialModules = Array.from({ length: 30 }, (_, index) => ({
  id: `module-${index + 1}`,
  content: `Module ${index + 1}`,
}));

// Start with empty user items
const initialUsers = [
  { id: "1", content: "Admin", items: [] },
  { id: "2", content: "Faculty", items: [] },
  { id: "3", content: "Mentor", items: [] },
];

export default function ModuleUp() {
  const [modules, setModules] = useState(initialModules);
  const [users, setUsers] = useState(initialUsers);

  const onDragEnd = (result) => {
    const { source, destination } = result;

    // If dropped outside the droppable area, do nothing
    if (!destination) return;

    // If the item was dropped in the same place, do nothing
    if (source.droppableId === destination.droppableId && source.index === destination.index) {
      return;
    }

    // Case where item is moved from the "modules" area to a user
    if (source.droppableId === "modules" && destination.droppableId !== "modules") {
      const sourceModules = [...modules];
      const destinationUserIndex = users.findIndex((user) => user.id === destination.droppableId);
      const destinationUser = { ...users[destinationUserIndex] };

      // Remove the dragged item from the "modules" section
      const [movedModule] = sourceModules.splice(source.index, 1);

      // Add the moved module to the destination user's items
      destinationUser.items.splice(destination.index, 0, movedModule);

      const newModules = sourceModules;
      const newUsers = [...users];
      newUsers[destinationUserIndex] = destinationUser;

      setModules(newModules);
      setUsers(newUsers); // Update the users' state
    }

    // Case where item is moved within a user's own section
    if (source.droppableId !== "modules" && destination.droppableId !== "modules") {
      const sourceUserIndex = users.findIndex((user) => user.id === source.droppableId);
      const destinationUserIndex = users.findIndex((user) => user.id === destination.droppableId);

      const sourceUser = { ...users[sourceUserIndex] };
      const destinationUser = { ...users[destinationUserIndex] };

      // Remove the dragged item from the source user's items
      const [movedModule] = sourceUser.items.splice(source.index, 1);

      // Add the moved module to the destination user's items
      destinationUser.items.splice(destination.index, 0, movedModule);

      const newUsers = [...users];
      newUsers[sourceUserIndex] = sourceUser;
      newUsers[destinationUserIndex] = destinationUser;

      setUsers(newUsers); // Update the users' state
    }
  };

  return (
    <section
      style={{
        display: "flex",
        justifyContent: "space-between",
        backgroundColor: "#f6f6f6",
        height: "100vh",
        padding: "20px",
      }}
    >
      <DragDropContext onDragEnd={onDragEnd}>
        {/* Modules List */}
        <Box
          sx={{
            padding: 2,
            borderRadius: "4px",
            backgroundColor: "white",
            width: "310px",
            height: "90vh",
            boxShadow: "rgba(0,0,0,0.2) 0px 0px 5px",
            overflowY: "scroll", // Scroll when too many modules
            paddingBottom: "10px",
          }}
        >
          <Typography variant="h6" component="h6">
            List of Modules
          </Typography>
          <Droppable droppableId="modules">
            {(provided) => (
              <Box
                ref={provided.innerRef}
                {...provided.droppableProps}
                sx={{ marginTop: "15px" }}
              >
                {modules.map((module, index) => (
                  <Draggable key={module.id} draggableId={module.id} index={index}>
                    {(provided) => (
                      <Box
                        ref={provided.innerRef}
                        {...provided.draggableProps}
                        {...provided.dragHandleProps}
                        sx={{
                          padding: 1,
                          marginBottom: 2,
                          backgroundColor: "#f3f3f3",
                          borderRadius: "4px",
                          boxShadow: "0 2px 5px rgba(0,0,0,0.2)",
                        }}
                      >
                        {module.content}
                      </Box>
                    )}
                  </Draggable>
                ))}
                {provided.placeholder}
              </Box>
            )}
          </Droppable>
        </Box>

        {/* Users Sections */}
        <Box
          sx={{
            padding: 2,
            borderRadius: "4px",
            backgroundColor: "white",
            height: "60vh",
            display: "flex",
            width: "100%",
            marginLeft: "50px",
          }}
        >
          {users.map((user) => (
            <Box
              key={user.id}
              sx={{
                padding: 2,
                borderRadius: "4px",
                backgroundColor: "white",
                width: "350px",  // Set to 350px width as per your requirement
                height: "600px", // Set to 600px height
                boxShadow: "rgba(0,0,0,0.2) 0px 0px 5px",
                margin: "10px",
                overflowY: "auto",  // Allow scrolling when there are many items
              }}
            >
              <Typography variant="h6" component="h6">
                List of {user.content} Modules
              </Typography>
              <Droppable droppableId={user.id}>
                {(provided) => (
                  <Box
                    ref={provided.innerRef}
                    {...provided.droppableProps}
                    sx={{
                      marginTop: "15px",
                      height: "100%", // Take the full height of the container
                      overflowY: "auto", // Allow scrolling inside the droppable area
                    }}
                  >
                    {user.items.map((module, index) => (
                      <Draggable key={module.id} draggableId={module.id} index={index}>
                        {(provided) => (
                          <Box
                            ref={provided.innerRef}
                            {...provided.draggableProps}
                            {...provided.dragHandleProps}
                            sx={{
                              padding: 1,
                              marginBottom: 2,
                              backgroundColor: "#f3f3f3",
                              borderRadius: "4px",
                              boxShadow: "0 2px 5px rgba(0,0,0,0.2)",
                            }}
                          >
                            {module.content}
                          </Box>
                        )}
                      </Draggable>
                    ))}
                    {provided.placeholder}
                  </Box>
                )}
              </Droppable>
            </Box>
          ))}
        </Box>
      </DragDropContext>
    </section>
  );
}
```

## Explanation:

Sure! I'll explain the code step by step so you can understand how to write similar code yourself. This code creates a drag-and-drop interface using `react-beautiful-dnd` to manage a list of modules that can be dragged and dropped into different user sections (like Admin, Faculty, and Mentor). Here's the breakdown:

### 1. **Importing Dependencies**

```javascript
import React, { useState } from "react";
import { DragDropContext, Draggable, Droppable } from "react-beautiful-dnd";
import { Box, Typography } from "@mui/material";
```

- **`React` and `useState`**: We're using React to build the component, and `useState` to manage the component's state.
- **`react-beautiful-dnd`**: This is the main library for implementing drag-and-drop functionality. It provides components like `DragDropContext`, `Draggable`, and `Droppable` to manage draggable elements and drop targets.
- **`@mui/material`**: This is Material-UI, a popular UI library. We're using `Box` and `Typography` components from it for styling.

### 2. **Initial Data Setup**

```javascript
const initialModules = Array.from({ length: 30 }, (_, index) => ({
  id: `module-${index + 1}`,
  content: `Module ${index + 1}`,
}));

const initialUsers = [
  { id: "1", content: "Admin", items: [] },
  { id: "2", content: "Faculty", items: [] },
  { id: "3", content: "Mentor", items: [] },
];
```

- **`initialModules`**: This is an array of 30 modules, each with a unique `id` and `content` (e.g., "Module 1", "Module 2", etc.). We're using `Array.from` to generate this array dynamically.
- **`initialUsers`**: This is an array of users (Admin, Faculty, and Mentor). Each user has an `id`, a `content` field (the user type), and an empty `items` array where modules will be stored.

### 3. **The Main Component (`ModuleUp`)**

```javascript
export default function ModuleUp() {
  const [modules, setModules] = useState(initialModules);
  const [users, setUsers] = useState(initialUsers);
```

- **`ModuleUp`** is the main component that manages the drag-and-drop functionality.
- **State**: We use `useState` to create two pieces of state:
  - `modules`: This holds the list of available modules.
  - `users`: This holds the list of users and their respective modules.

### 4. **Handling the Drag and Drop (`onDragEnd`)**

```javascript
const onDragEnd = (result) => {
  const { source, destination } = result;

  if (!destination) return;

  if (
    source.droppableId === destination.droppableId &&
    source.index === destination.index
  ) {
    return;
  }

  if (
    source.droppableId === "modules" &&
    destination.droppableId !== "modules"
  ) {
    // Handle module moved to a user section
  }

  if (
    source.droppableId !== "modules" &&
    destination.droppableId !== "modules"
  ) {
    // Handle module moved within a user section
  }
};
```

- **`onDragEnd`**: This function is triggered when the user finishes dragging an item.
  - **`source` and `destination`**: These contain information about where the item is dragged from and where it is dropped.
  - **Checking conditions**:
    - If the item was dropped outside the droppable area (i.e., there’s no destination), we return early.
    - If the item was dropped back in the same position, we also do nothing.

#### Case 1: **Moving a Module from the `modules` List to a User**

```javascript
if (source.droppableId === "modules" && destination.droppableId !== "modules") {
  const sourceModules = [...modules];
  const destinationUserIndex = users.findIndex(
    (user) => user.id === destination.droppableId
  );
  const destinationUser = { ...users[destinationUserIndex] };

  const [movedModule] = sourceModules.splice(source.index, 1);

  destinationUser.items.splice(destination.index, 0, movedModule);

  const newModules = sourceModules;
  const newUsers = [...users];
  newUsers[destinationUserIndex] = destinationUser;

  setModules(newModules);
  setUsers(newUsers);
}
```

- **Moving from `modules` to a user**:
  - Find the user (Admin, Faculty, etc.) where the module is dropped.
  - Remove the module from the `modules` list and add it to the selected user's `items` array.
  - Update the state using `setModules` and `setUsers`.

#### Case 2: **Moving a Module Between Users**

```javascript
if (source.droppableId !== "modules" && destination.droppableId !== "modules") {
  const sourceUserIndex = users.findIndex(
    (user) => user.id === source.droppableId
  );
  const destinationUserIndex = users.findIndex(
    (user) => user.id === destination.droppableId
  );

  const sourceUser = { ...users[sourceUserIndex] };
  const destinationUser = { ...users[destinationUserIndex] };

  const [movedModule] = sourceUser.items.splice(source.index, 1);

  destinationUser.items.splice(destination.index, 0, movedModule);

  const newUsers = [...users];
  newUsers[sourceUserIndex] = sourceUser;
  newUsers[destinationUserIndex] = destinationUser;

  setUsers(newUsers);
}
```

- **Moving within a user’s own section**:
  - Find the source and destination users.
  - Remove the module from the source user's list and add it to the destination user’s list.
  - Update the state with the modified user lists.

### 5. **Rendering the UI**

```javascript
return (
  <section
    style={{
      display: "flex",
      justifyContent: "space-between",
      backgroundColor: "#f6f6f6",
      height: "100vh",
      padding: "20px",
    }}
  >
    <DragDropContext onDragEnd={onDragEnd}>
      {/* Modules List */}
      <Box
        sx={{
          padding: 2,
          borderRadius: "4px",
          backgroundColor: "white",
          width: "310px",
          height: "90vh",
          boxShadow: "rgba(0,0,0,0.2) 0px 0px 5px",
          overflowY: "scroll",
          paddingBottom: "10px",
        }}
      >
        <Typography variant="h6" component="h6">
          List of Modules
        </Typography>
        <Droppable droppableId="modules">
          {(provided) => (
            <Box
              ref={provided.innerRef}
              {...provided.droppableProps}
              sx={{ marginTop: "15px" }}
            >
              {modules.map((module, index) => (
                <Draggable
                  key={module.id}
                  draggableId={module.id}
                  index={index}
                >
                  {(provided) => (
                    <Box
                      ref={provided.innerRef}
                      {...provided.draggableProps}
                      {...provided.dragHandleProps}
                      sx={{
                        padding: 1,
                        marginBottom: 2,
                        backgroundColor: "#f3f3f3",
                        borderRadius: "4px",
                        boxShadow: "0 2px 5px rgba(0,0,0,0.2)",
                      }}
                    >
                      {module.content}
                    </Box>
                  )}
                </Draggable>
              ))}
              {provided.placeholder}
            </Box>
          )}
        </Droppable>
      </Box>

      {/* Users Sections */}
      <Box
        sx={{
          padding: 2,
          borderRadius: "4px",
          backgroundColor: "white",
          height: "60vh",
          display: "flex",
          width: "100%",
          marginLeft: "50px",
        }}
      >
        {users.map((user) => (
          <Box
            key={user.id}
            sx={{
              padding: 2,
              borderRadius: "4px",
              backgroundColor: "white",
              width: "350px",
              height: "600px",
              boxShadow: "rgba(0,0,0,0.2) 0px 0px 5px",
              margin: "10px",
              overflowY: "auto",
            }}
          >
            <Typography variant="h6" component="h6">
              List of {user.content} Modules
            </Typography>
            <Droppable droppableId={user.id}>
              {(provided) => (
                <Box
                  ref={provided.innerRef}
                  {...provided.droppableProps}
                  sx={{ marginTop: "15px", height: "100%", overflowY: "auto" }}
                >
                  {user.items.map((module, index) => (
                    <Draggable
                      key={module.id}
                      draggableId={module.id}
                      index={index}
                    >
                      {(provided) => (
                        <Box
                          ref={provided.innerRef}
                          {...provided.draggableProps}
                          {...provided.dragHandleProps}
                          sx={{
                            padding: 1,
                            marginBottom: 2,
                            backgroundColor: "#f3f3f3",
                            borderRadius: "4px",
                            boxShadow: "0 2px 5px rgba(0,0,0,0.2)",
                          }}
                        >
                          {module.content}
                        </Box>
                      )}
                    </Draggable>
                  ))}
                  {provided.placeholder}
                </Box>
              )}
            </Droppable>
          </Box>
        ))}
      </Box>
    </DragDropContext>
  </section>
);
```

- The main layout is structured using `Box` components for the **Modules List** and \*\*

Users Sections\*\*.

- **`DragDropContext`** wraps the entire drag-and-drop area and receives `onDragEnd`.
- Inside, **`Droppable`** defines areas where draggable items can be dropped.
  - **Modules List**: The left column where modules can be moved from.
  - **Users Sections**: The right column where users can receive the modules.

### Summary of Key Concepts:

- **State Management**: We manage the list of modules and users using `useState`.
- **Drag-and-Drop**: The `DragDropContext`, `Droppable`, and `Draggable` components enable drag-and-drop behavior.
- **Updating State**: After an item is moved, we update the state for modules or users using `setModules` or `setUsers` based on where the item is dropped.
