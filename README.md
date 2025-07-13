# Laberinto
import React, { useRef, useEffect, useState, useCallback } from 'react';
import * as THREE from 'three';

// Main App component for the 3D maze game
const App = () => {
    // Refs for the canvas element and game state
    const mountRef = useRef(null); // Main game canvas mount point
    const sceneRef = useRef(null);
    const cameraRef = useRef(null);
    const rendererRef = useRef(null);

    const playerRef = useRef(new THREE.Vector3(0, 0, 0)); // Player position
    const playerVelocityRef = useRef(new THREE.Vector3(0, 0, 0)); // Player velocity for movement
    const playerDirectionRef = useRef(new THREE.Vector3(0, 0, -1)); // Player looking direction
    const playerRotationRef = useRef(new THREE.Euler(0, 0, 0, 'YXZ')); // Player camera rotation
    const mazeRef = useRef([]); // 2D array representing the maze
    const wallsRef = useRef([]); // Array of Three.js wall meshes
    const exitRef = useRef(new THREE.Vector3(0, 0, 0)); // Exit position
    const floorRef = useRef(null); // Ref for the floor mesh
    const collectiblesRef = useRef([]); // Array of collectible objects in main scene
    const clueMarkersRef = useRef([]); // Array of clue markers

    // State for movement controls (now managed via ref for animation loop)
    const movementStateRef = useRef({
        moveForward: false,
        moveBackward: false,
        moveLeft: false,
        moveRight: false,
    });

    // State for lever dragging
    const isDraggingLeverRef = useRef(false); // New ref to track if the lever is being dragged

    const [gameStarted, setGameStarted] = useState(false); // State to control game start
    const [gameWon, setGameWon] = useState(false); // State for win condition
    const [showInstructions, setShowInstructions] = useState(true); // State to show/hide instructions
    const [threeJsInitialized, setThreeJsInitialized] = useState(false); // State for Three.js initialization status
    const [collectedItemsCount, setCollectedItemsCount] = useState(0); // Counter for collected items
    const [totalItemsCount, setTotalItemsCount] = useState(0); // Total number of items to collect

    // Constants for game parameters
    const MAZE_SIZE = 21; // Maze size
    const WALL_HEIGHT = 5;
    const WALL_WIDTH = 5;
    const PLAYER_HEIGHT = WALL_HEIGHT / 2;
    const MOVEMENT_SPEED = 0.08; // Reduced speed for a slower trot
    const MOUSE_SENSITIVITY = 0.005; // Sensitivity for camera rotation with the lever
    const PLAYER_RADIUS = 1.5; // For collision detection
    const COLLECTIBLE_RADIUS = 1; // Radius for collectible collision
    const CLUE_MARKER_SIZE = 0.5; // Size of clue markers
    const NUM_COLLECTIBLES = 15; // Fixed number of collectibles

    // Define different types of collectibles
    const collectibleTypes = [
        { name: 'Gema', geometry: new THREE.SphereGeometry(COLLECTIBLE_RADIUS, 16, 16), material: new THREE.MeshBasicMaterial({ color: 0x00ffcc }) }, // Cyan glow
        { name: 'Pergamino', geometry: new THREE.CylinderGeometry(COLLECTIBLE_RADIUS * 0.5, COLLECTIBLE_RADIUS * 0.5, COLLECTIBLE_RADIUS * 1.5, 8), material: new THREE.MeshBasicMaterial({ color: 0xffff00 }) }, // Yellow glow
        { name: 'Anillo', geometry: new THREE.TorusGeometry(COLLECTIBLE_RADIUS * 0.7, COLLECTIBLE_RADIUS * 0.2, 8, 16), material: new THREE.MeshBasicMaterial({ color: 0xff66ff }) }, // Magenta glow
        { name: 'Cristal', geometry: new THREE.OctahedronGeometry(COLLECTIBLE_RADIUS), material: new THREE.MeshBasicMaterial({ color: 0x0099ff }) }, // Blue glow
    ];

    // Define single wall material
    const wallMaterial = useRef(new THREE.MeshStandardMaterial({ color: 0x333333, roughness: 0.8, metalness: 0.2 })); // Dark Gray
    // Material for clues (glowing orange)
    const clueMaterial = useRef(new THREE.MeshBasicMaterial({ color: 0xffa500, emissive: 0xffa500, emissiveIntensity: 0.8 })); // Orange glow

    // Function to generate the maze structure and add walls to scene
    const generateMaze = useCallback((size, scene) => {
        const grid = Array(size).fill(0).map(() => Array(size).fill(1)); // 1 for wall, 0 for path

        // Helper function to check if a cell is valid
        const isValid = (x, y) => x >= 0 && x < size && y >= 0 && y < size;

        // Recursive function for maze generation
        const carvePath = (cx, cy) => {
            if (!isValid(cx, cy)) {
                console.error(`Invalid cell coordinates for carving: (${cx}, ${cy})`);
                return;
            }
            grid[cy][cx] = 0; // Mark current cell as path

            const directions = [
                [0, -2], // Up
                [0, 2],  // Down
                [-2, 0], // Left
                [2, 0]   // Right
            ];

            // Shuffle directions to ensure random maze generation
            for (let i = directions.length - 1; i > 0; i--) {
                const j = Math.floor(Math.random() * (i + 1));
                [directions[i], directions[j]] = [directions[j], directions[i]];
            }

            for (const [dx, dy] of directions) {
                const nx = cx + dx;
                const ny = cy + dy;
                const wallX = cx + dx / 2;
                const wallY = cy + dy / 2;

                if (isValid(nx, ny) && grid[ny][nx] === 1) {
                    if (isValid(wallX, wallY)) {
                        grid[wallY][wallX] = 0; // Carve path through the wall
                        carvePath(nx, ny);
                    } else {
                        console.warn(`Attempted to carve through invalid wall coordinates: (${wallX}, ${wallY}) for neighbor (${nx}, ${ny}) from (${cx}, ${cy})`);
                    }
                }
            }
        };

        const maxOffset = Math.floor((size - 2) / 2);
        const startX = Math.floor(Math.random() * (maxOffset + 1)) * 2 + 1;
        const startY = Math.floor(Math.random() * (maxOffset + 1)) * 2 + 1;
        
        if (startX < 1 || startX >= size -1 || startY < 1 || startY >= size -1) {
             console.error("Calculated start position is out of bounds. Adjusting to (1,1).");
             carvePath(1, 1);
        } else {
            carvePath(startX, startY);
        }

        playerRef.current.set((startX - size / 2 + 0.5) * WALL_WIDTH, PLAYER_HEIGHT, (startY - size / 2 + 0.5) * WALL_WIDTH);

        const exitX = size - 2;
        const exitY = size - 2;
        grid[exitY][exitX] = 0; // Ensure exit is a path
        exitRef.current.set((exitX - size / 2 + 0.5) * WALL_WIDTH, PLAYER_HEIGHT, (exitY - size / 2 + 0.5) * WALL_WIDTH);

        grid[startY][startX] = 2; // Start
        grid[exitY][exitX] = 3; // Exit

        mazeRef.current = grid;

        // --- Add walls to scene ---
        const wallGeometry = new THREE.BoxGeometry(WALL_WIDTH, WALL_HEIGHT, WALL_WIDTH);
        const clueMarkerGeometry = new THREE.BoxGeometry(CLUE_MARKER_SIZE, CLUE_MARKER_SIZE, CLUE_MARKER_SIZE); // Small cube for clues
        
        wallsRef.current = []; // Clear previous walls
        clueMarkersRef.current = []; // Clear previous clue markers

        for (let y = 0; y < MAZE_SIZE; y++) {
            for (let x = 0; x < MAZE_SIZE; x++) {
                if (grid[y][x] === 1) { // If it's a wall
                    const wall = new THREE.Mesh(wallGeometry, wallMaterial.current);
                    wall.position.set((x - MAZE_SIZE / 2 + 0.5) * WALL_WIDTH, WALL_HEIGHT / 2, (y - MAZE_SIZE / 2 + 0.5) * WALL_WIDTH);
                    scene.add(wall);
                    wallsRef.current.push(wall);

                    // Add a clue marker to some walls (not outer boundary, not too close to start/exit)
                    // Ensure all array accesses are within bounds
                    if (Math.random() < 0.03 && x > 0 && x < size - 1 && y > 0 && y < size - 1) {
                        // Check if the adjacent cells are valid before accessing them
                        const hasHorizontalPath = isValid(x - 1, y) && grid[y][x-1] === 0 && isValid(x + 1, y) && grid[y][x+1] === 0;
                        const hasVerticalPath = isValid(x, y - 1) && grid[y-1][x] === 0 && isValid(x, y + 1) && grid[y+1][x] === 0;

                        if (hasHorizontalPath || hasVerticalPath) {
                            const clueMarker = new THREE.Mesh(clueMarkerGeometry, clueMaterial.current);
                            clueMarker.position.copy(wall.position);
                            clueMarker.position.y = WALL_HEIGHT * 0.75; // Place slightly higher on the wall
                            scene.add(clueMarker);
                            clueMarkersRef.current.push(clueMarker);
                        }
                    }
                }
            }
        }
        // Dispose geometries created within this function after they are used to create meshes
        wallGeometry.dispose();
        clueMarkerGeometry.dispose();

        return grid;
    }, []);

    // Function to place collectibles in the maze
    const placeCollectibles = useCallback((mazeGrid, scene) => {
        collectiblesRef.current = []; // Clear previous collectibles
        let potentialCollectiblePositions = [];

        for (let y = 0; y < MAZE_SIZE; y++) {
            for (let x = 0; x < MAZE_SIZE; x++) {
                // Collect all valid path cells (0) that are not the start (2) or exit (3)
                if (mazeGrid[y][x] === 0) {
                    potentialCollectiblePositions.push({ x, y });
                }
            }
        }

        // Shuffle the potential positions
        for (let i = potentialCollectiblePositions.length - 1; i > 0; i--) {
            const j = Math.floor(Math.random() * (i + 1));
            [potentialCollectiblePositions[i], potentialCollectiblePositions[j]] = [potentialCollectiblePositions[j], potentialCollectiblePositions[i]];
        }

        // Place exactly NUM_COLLECTIBLES, or fewer if not enough positions are available
        const numToPlace = Math.min(NUM_COLLECTIBLES, potentialCollectiblePositions.length);
        
        for (let i = 0; i < numToPlace; i++) {
            const { x, y } = potentialCollectiblePositions[i];
            const typeIndex = Math.floor(Math.random() * collectibleTypes.length);
            const collectibleType = collectibleTypes[typeIndex];

            const collectibleMesh = new THREE.Mesh(collectibleType.geometry, collectibleType.material);
            collectibleMesh.position.set(
                (x - MAZE_SIZE / 2 + 0.5) * WALL_WIDTH,
                PLAYER_HEIGHT, // Place at player height
                (y - MAZE_SIZE / 2 + 0.5) * WALL_WIDTH
            );
            // Add a unique identifier for collision detection and removal
            collectibleMesh.userData = { isCollectible: true, id: `collectible_${x}_${y}_${typeIndex}` };
            scene.add(collectibleMesh);
            collectiblesRef.current.push(collectibleMesh);
        }
        setTotalItemsCount(numToPlace); // Set total items to the actual number placed
        setCollectedItemsCount(0); // Reset collected count on new maze generation
        console.log("Total collectibles placed:", numToPlace); // Log the count
    }, [collectibleTypes]);

    // Effect for initializing Three.js scene, camera, renderer, and lights
    useEffect(() => {
        if (!mountRef.current) {
            console.log("Three.js init useEffect: mountRef not yet available, returning.");
            return;
        }
        console.log("Three.js init useEffect: mountRef is available. Proceeding with initialization.");

        // --- Main Scene Setup ---
        const scene = new THREE.Scene();
        sceneRef.current = scene;

        const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
        camera.position.set(playerRef.current.x, playerRef.current.y, playerRef.current.z);
        cameraRef.current = camera;

        const renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(window.innerWidth, window.innerHeight);
        renderer.setClearColor(0x1a0a0a); // Very dark red/brown for terror
        mountRef.current.appendChild(renderer.domElement);
        rendererRef.current = renderer;

        // Lighting
        const ambientLight = new THREE.AmbientLight(0xffffff, 1.5); // Much brighter ambient light
        scene.add(ambientLight);

        const pointLight = new THREE.PointLight(0xffffff, 3, 100); // Very strong flashlight, longer range
        camera.add(pointLight); // Attach light to camera
        scene.add(camera); // Add camera to scene so light attached to it is rendered

        // Fog for limited visibility and creepy atmosphere
        scene.fog = new THREE.Fog(0x1a0a0a, 50, 150); // Much less dense fog, starts further away

        // Add floor to the main scene (persistent)
        const floorGeometry = new THREE.PlaneGeometry(MAZE_SIZE * WALL_WIDTH, MAZE_SIZE * WALL_WIDTH);
        const floorMaterial = new THREE.MeshStandardMaterial({ color: 0x111111, roughness: 0.9, metalness: 0.1, side: THREE.DoubleSide }); // Darker, rougher floor
        const floor = new THREE.Mesh(floorGeometry, floorMaterial);
        floor.rotation.x = -Math.PI / 2; // Rotate to be horizontal
        floor.position.y = 0; // Position at the base of the walls
        scene.add(floor);
        floorRef.current = floor;

        // Handle window resize
        const handleResize = () => {
            if (cameraRef.current && rendererRef.current) {
                cameraRef.current.aspect = window.innerWidth / window.innerHeight;
                cameraRef.current.updateProjectionMatrix();
                rendererRef.current.setSize(window.innerWidth, window.innerHeight);
            }
        };
        window.addEventListener('resize', handleResize);

        // Set Three.js as initialized
        setThreeJsInitialized(true);
        console.log("Three.js initialized state set to true.");

        // Cleanup function
        return () => {
            console.log("Three.js init useEffect cleanup running.");
            window.removeEventListener('resize', handleResize);
            if (mountRef.current && rendererRef.current && rendererRef.current.domElement) {
                mountRef.current.removeChild(rendererRef.current.domElement);
            }
            // Dispose of Three.js objects to prevent memory leaks
            scene.traverse((object) => {
                if (object.isMesh) {
                    object.geometry.dispose();
                    object.material.dispose();
                }
            });
            renderer.dispose();
            floorGeometry.dispose();
            floorMaterial.dispose();
            wallMaterial.current.dispose(); // Dispose the single wall material
            clueMaterial.current.dispose(); // Dispose clue material
        };
    }, []); // Empty dependency array: runs only once on mount

    // Function to start the game (now a useCallback)
    const startGame = useCallback(() => {
        // Ensure Three.js is initialized before proceeding
        if (!threeJsInitialized || !sceneRef.current || !rendererRef.current) {
            console.warn("startGame: Three.js not fully initialized yet. Cannot start game.");
            return;
        }
        console.log("startGame called. sceneRef.current:", sceneRef.current);

        setGameStarted(true);
        setGameWon(false);
        setShowInstructions(false); // Ensure instructions are hidden

        // --- Game Setup on Start/Restart ---
        // Clear existing maze and walls from scene
        if (sceneRef.current) {
            wallsRef.current.forEach(wall => sceneRef.current.remove(wall));
            wallsRef.current = [];
            // Remove exit marker if it exists from main scene
            const exitMarkerMain = sceneRef.current.children.find(obj => obj.geometry instanceof THREE.SphereGeometry && obj.material.color.getHex() === 0x00ff00);
            if (exitMarkerMain) sceneRef.current.remove(exitMarkerMain);
            // Remove all collectibles from main scene
            collectiblesRef.current.forEach(collectible => sceneRef.current.remove(collectible));
            collectiblesRef.current = [];
            // Remove all clue markers
            clueMarkersRef.current.forEach(marker => sceneRef.current.remove(marker));
            clueMarkersRef.current = [];
        }

        // Re-generate maze structure and add walls to scene
        const mazeGrid = generateMaze(MAZE_SIZE, sceneRef.current);
        
        // Re-add exit marker to main scene
        const exitMarkerGeometry = new THREE.SphereGeometry(1, 32, 32);
        const exitMarkerMaterial = new THREE.MeshBasicMaterial({ color: 0x00ff00, transparent: true, opacity: 0.5 });
        const exitMarker = new THREE.Mesh(exitMarkerGeometry, exitMarkerMaterial);
        exitMarker.position.set(exitRef.current.x, exitRef.current.y, exitRef.current.z);
        sceneRef.current.add(exitMarker);
        exitMarkerGeometry.dispose(); // Dispose geometry after use
        exitMarkerMaterial.dispose(); // Dispose material after use

        // Place new collectibles for the game
        placeCollectibles(mazeGrid, sceneRef.current);


        // Reset camera and player rotation
        playerRotationRef.current.set(0, 0, 0, 'YXZ');
        if (cameraRef.current) {
            cameraRef.current.rotation.copy(playerRotationRef.current);
            cameraRef.current.position.copy(playerRef.current);
        }
    }, [threeJsInitialized, sceneRef, rendererRef, generateMaze, placeCollectibles]);


    // Effect for handling keyboard input (movement)
    useEffect(() => {
        const handleKeyDown = (event) => {
            switch (event.code) {
                case 'KeyW': movementStateRef.current.moveForward = true; break;
                case 'KeyS': movementStateRef.current.moveBackward = true; break;
                case 'KeyA': movementStateRef.current.moveLeft = true; break;
                case 'KeyD': movementStateRef.current.moveRight = true; break;
                default: break;
            }
        };

        const handleKeyUp = (event) => {
            switch (event.code) {
                case 'KeyW': movementStateRef.current.moveForward = false; break;
                case 'KeyS': movementStateRef.current.moveBackward = false; break;
                case 'KeyA': movementStateRef.current.moveLeft = false; break;
                case 'KeyD': movementStateRef.current.moveRight = false; break;
                default: break;
            }
        };

        window.addEventListener('keydown', handleKeyDown);
        window.addEventListener('keyup', handleKeyUp);

        return () => {
            window.removeEventListener('keydown', handleKeyDown);
            window.removeEventListener('keyup', handleKeyUp);
        };
    }, []); // Empty dependency array, runs once

    // Mouse handlers for the new joystick/lever
    const handleLeverMouseDown = useCallback(() => {
        if (gameStarted) {
            isDraggingLeverRef.current = true;
        }
    }, [gameStarted]);

    const handleLeverMouseMove = useCallback((event) => {
        if (!isDraggingLeverRef.current || !cameraRef.current || !playerRotationRef.current) {
            return;
        }

        const movementX = event.movementX || event.mozMovementX || event.webkitMovementX || 0;
        const movementY = event.movementY || event.mozMovementY || event.webkitMovementY || 0;

        if (movementX !== 0 || movementY !== 0) {
            playerRotationRef.current.y -= movementX * MOUSE_SENSITIVITY;
            playerRotationRef.current.x -= movementY * MOUSE_SENSITIVITY;

            playerRotationRef.current.x = Math.max(-Math.PI / 2, Math.min(Math.PI / 2, playerRotationRef.current.x));

            cameraRef.current.rotation.copy(playerRotationRef.current);
        }
    }, []);

    const handleLeverMouseUp = useCallback(() => {
        isDraggingLeverRef.current = false;
    }, []);

    // Effect to attach and detach mouse listeners for the lever
    useEffect(() => {
        // Attach global mousemove and mouseup listeners to capture events even outside the lever area
        window.addEventListener('mousemove', handleLeverMouseMove);
        window.addEventListener('mouseup', handleLeverMouseUp);

        return () => {
            window.removeEventListener('mousemove', handleLeverMouseMove);
            window.removeEventListener('mouseup', handleLeverMouseUp);
        };
    }, [handleLeverMouseMove, handleLeverMouseUp]); // Dependencies ensure handlers are up-to-date

    // Animation loop
    useEffect(() => {
        let animationFrameId;

        const animate = () => {
            // Ensure Three.js components are initialized before proceeding
            if (!rendererRef.current || !sceneRef.current || !cameraRef.current) {
                animationFrameId = requestAnimationFrame(animate); // Keep trying until initialized
                return;
            }

            // Render the main scene
            rendererRef.current.render(sceneRef.current, cameraRef.current);

            // Only update game logic if game has started and not won
            if (gameStarted && !gameWon) {
                const camera = cameraRef.current;
                const playerPosition = playerRef.current;
                const playerVelocity = playerVelocityRef.current;
                const playerDirection = playerDirectionRef.current;
                const { moveForward, moveBackward, moveLeft, moveRight } = movementStateRef.current; // Get latest state from ref

                // Update player direction based on camera rotation
                playerDirection.set(0, 0, -1).applyEuler(playerRotationRef.current);

                // Calculate movement vector
                const forward = new THREE.Vector3().setFromMatrixColumn(camera.matrix, 0); // Right vector
                forward.crossVectors(camera.up, forward); // Forward vector
                forward.y = 0; // Keep movement on horizontal plane
                forward.normalize();

                const right = new THREE.Vector3().setFromMatrixColumn(camera.matrix, 0); // Right vector
                right.y = 0; // Keep movement on horizontal plane
                right.normalize();

                playerVelocity.set(0, 0, 0);

                if (moveForward) playerVelocity.add(forward);
                if (moveBackward) playerVelocity.sub(forward);
                if (moveLeft) playerVelocity.sub(right);
                if (moveRight) playerVelocity.add(right);

                playerVelocity.normalize().multiplyScalar(MOVEMENT_SPEED);

                // Proposed new position
                const newPlayerPosition = playerPosition.clone().add(playerVelocity);

                // Collision detection with walls
                let collided = false;
                for (const wall of wallsRef.current) {
                    // Simple AABB collision detection for player (sphere) and wall (box)
                    // Convert player position to local coordinates of the wall
                    const localPlayerPos = wall.worldToLocal(newPlayerPosition.clone());

                    // Check if player's sphere collides with the wall's bounding box
                    const halfWallX = WALL_WIDTH / 2;
                    const halfWallY = WALL_HEIGHT / 2;
                    const halfWallZ = WALL_WIDTH / 2;

                    const closestX = Math.max(-halfWallX, Math.min(localPlayerPos.x, halfWallX));
                    const closestY = Math.max(-halfWallY, Math.min(localPlayerPos.y, halfWallY));
                    const closestZ = Math.max(-halfWallZ, Math.min(localPlayerPos.z, halfWallZ));

                    const deltaX = localPlayerPos.x - closestX;
                    const deltaY = localPlayerPos.y - closestY;
                    const deltaZ = localPlayerPos.z - closestZ;

                    const distanceSq = (deltaX * deltaX) + (deltaY * deltaY) + (deltaZ * deltaZ);

                    if (distanceSq < (PLAYER_RADIUS * PLAYER_RADIUS)) {
                        collided = true;
                        break;
                    }
                }

                if (!collided) {
                    playerPosition.copy(newPlayerPosition);
                    camera.position.copy(playerPosition);
                }

                // Collision detection with collectibles
                collectiblesRef.current = collectiblesRef.current.filter(collectible => {
                    if (playerPosition.distanceTo(collectible.position) < PLAYER_RADIUS + COLLECTIBLE_RADIUS) {
                        sceneRef.current.remove(collectible); // Remove from scene
                        setCollectedItemsCount(prevCount => prevCount + 1); // Update count
                        return false; // Remove from active collectibles list
                    }
                    return true; // Keep in list
                });


                // Check for win condition (now includes collecting all items)
                if (playerPosition.distanceTo(exitRef.current) < PLAYER_RADIUS + 1 && collectedItemsCount === totalItemsCount && totalItemsCount > 0) {
                    setGameWon(true);
                }
            }

            animationFrameId = requestAnimationFrame(animate); // Request next frame
        };

        // Start animation loop only once when component mounts
        animationFrameId = requestAnimationFrame(animate);

        // Cleanup: cancel animation frame when component unmounts
        return () => cancelAnimationFrame(animationFrameId);
    }, [gameStarted, gameWon, collectedItemsCount, totalItemsCount, threeJsInitialized]);


    // Function to restart the game
    const restartGame = () => {
        // This function now just calls startGame to re-initialize everything
        startGame();
    };

    return (
        <div className="relative w-screen h-screen overflow-hidden bg-gray-900 font-inter text-white">
            {/* Canvas for Three.js rendering */}
            <div ref={mountRef} className="absolute inset-0 z-0">
            </div>

            {/* Game UI */}
            <div className="absolute inset-0 flex flex-col items-center justify-center z-10 p-4">
                {showInstructions && (
                    <div className="bg-gray-800 bg-opacity-90 p-8 rounded-xl shadow-lg border border-gray-700 max-w-lg text-center">
                        <h1 className="text-4xl font-bold mb-4 text-red-400">Laberinto de la Oscuridad</h1>
                        {/* Simplified instructions: removed the detailed list */}
                        <p className="text-lg mb-4">
                            ¡Encuentra la salida y recolecta todos los objetos!
                        </p>
                        <button
                            onClick={startGame}
                            disabled={!threeJsInitialized} // Disable button until Three.js is ready
                            className={`font-bold py-3 px-8 rounded-full shadow-lg transition duration-300 ease-in-out transform hover:scale-105 focus:outline-none focus:ring-2 focus:ring-opacity-75 ${
                                threeJsInitialized ? 'bg-red-700 hover:bg-red-800 focus:ring-red-500' : 'bg-gray-500 cursor-not-allowed'
                            }`}
                        >
                            {threeJsInitialized ? 'Comenzar Aventura' : 'Cargando...'}
                        </button>
                    </div>
                )}

                {gameWon && (
                    <div className="bg-green-800 bg-opacity-90 p-8 rounded-xl shadow-lg border border-green-700 max-w-lg text-center">
                        <h1 className="text-4xl font-bold mb-4 text-green-300">¡Has Encontrado la Salida!</h1>
                        <p className="text-lg mb-6">
                            Felicitaciones, has escapado del laberinto oscuro.
                        </p>
                        <button
                            onClick={restartGame}
                            className="bg-blue-700 hover:bg-blue-800 text-white font-bold py-3 px-8 rounded-full shadow-lg transition duration-300 ease-in-out transform hover:scale-105 focus:outline-none focus:ring-2 focus:ring-blue-500 focus:ring-opacity-75"
                        >
                            Jugar de Nuevo
                        </button>
                    </div>
                )}

                {/* Collectible Counter */}
                {gameStarted && !gameWon && (
                    <div className="absolute top-8 left-1/2 -translate-x-1/2 bg-gray-800 bg-opacity-70 text-white text-lg py-2 px-6 rounded-lg shadow-md font-bold">
                        Objetos recolectados: {collectedItemsCount} / {totalItemsCount}
                    </div>
                )}

                {/* Joystick/Lever Area */}
                {gameStarted && !gameWon && (
                    <div
                        className="absolute bottom-8 right-8 w-32 h-32 bg-gray-700 bg-opacity-50 rounded-full flex items-center justify-center cursor-grab active:cursor-grabbing"
                        onMouseDown={handleLeverMouseDown}
                        // Mousemove and mouseup are handled globally on window
                    >
                        <div className="w-20 h-20 bg-gray-500 rounded-full shadow-lg flex items-center justify-center">
                            {/* Simple lever visual, could be an SVG or emoji */}
                            <span className="text-4xl">🕹️</span> {/* Joystick emoji as visual */}
                        </div>
                    </div>
                )}
            </div>
        </div>
    );
};

export default App;
