# circlepe-vidhan_A2305221542
👽 Intergalactic Trade Network - BE - CirclePe

-> Important: Detailed installation instructions and API documentation can be found in the /docs directory of this project.
🌟 Project Overview :

The Intergalactic Trade Network BE Circlepe is designed to support a sophisticated interplanetary trade network, managing trade transactions, space cargo logistics, and inventory at various space stations across the galaxy. Engineered for high throughput and real-time data processing, this system ensures efficient and seamless trading operations on a galactic scale.

🚀 Key Features ->

Trade Transactions: Efficiently manage and process trade operations between various entities across the galaxy. Space Cargo Management: Track and monitor the movement and status of cargo shipments between planets and space stations. Inventory Tracking: Maintain real-time updates and monitoring of inventory levels at space stations. Real-Time Updates: Provide immediate feedback and updates on trade activities and cargo movements.

🗄️ Project Structure
The project follows a modular structure, organizing code into different directories for clarity and maintainability.

src/
│
├── controllers/           # Contains controller files that define the business logic for each module
│   ├── trades.controller.ts  # Handles trade-related operations
│   ├── cargo.controller.ts  # Manages cargo operations
│   └── inventory.controller.ts  # Manages inventory operations
│   └── updates.controller.ts  # Manages real-time updates operations 
|
├── routes/                # Contains route files that map HTTP endpoints to the appropriate controllers
|   ├── index.ts            # Contains every router
│   ├── trade.routes.ts     # Routes for trade operations
│   ├── cargo.routes.ts     # Routes for cargo operations
│   └── inventory.routes.ts # Routes for inventory operations
│
├── events/                # Contains event emitters and listeners for real-time updates
│   ├── tradesEventEmitter.ts # Handles trade-related events
│   ├── cargoEventEmitter.ts  # Handles cargo-related events
│   └── eventEmitter.ts     #  Initialize event emitter
│
├── model/                 # Contains the Mongoose models representing the database schemas
│   ├── Trade.ts            # Trade model schema
│   ├── Cargo.ts            # Cargo model schema
│   └── Inventory.ts        # Inventory model schema
│
└── db/                    # Contains database connection and configuration files
    └── dbConnection.ts     # Sets up the connection to the MongoDB database
-> 🛤️ Routes
index.ts
The appRouter.ts file serves as the central routing hub for the backend system. It imports individual route modules and then associates them with specific base paths using the Express Router. This modular approach helps in organizing the application routes, making it easier to maintain and extend the system.

import { Router } from "express";
import tradeRouter from "./trade.routes";
import cargoRouter from "./cargo.routes";
import inventoryRouter from "./inventory.routes";
import updatesRouter from "./updates.routes";

const appRouter = Router();

appRouter.use("/trades", tradeRouter);
appRouter.use("/cargo", cargoRouter);
appRouter.use("/inventory", inventoryRouter);
appRouter.use("/updates", updatesRouter);

export default appRouter;

Overview of other Route Files in the Backend System 🗂️
In this backend system, each route file is designed to handle specific operations related to a particular domain, such as trade transactions, cargo management, or inventory tracking. This modular approach keeps the code organized, with each file focusing on a distinct set of endpoints and their corresponding operations.

Example: cargo.routes.ts
The cargo.routes.ts file is responsible for defining and handling routes related to cargo operations. Each route corresponds to a specific HTTP method and endpoint, designed to perform an action like retrieving, updating, or managing cargo data.


const cargoRouter = Router();

cargoRouter.get("/", getAllCargos);
cargoRouter.get("/:shipmentId", getCargoById);
cargoRouter.put("/update/:shipmentId", updateCargo);

export default cargoRouter;

How Routes Interact with Controllers in the Backend System
In this backend system, each route defined in the routing files (cargo.routes.ts, trade.routes.ts, etc.) is connected to a controller function located in the controllers directory. These controller functions handle the core operations, such as interacting with the database, processing data, and sending responses back to the client.

Example: cargo.controllers.ts function
export const updateCargo = async (req: Request, res: Response) => {
  const { shipmentId } = req.params;
  const updateData = req.body;

  try {
    const updatedCargo = await Cargo.findOneAndUpdate(
      { shipmentId },
      { $set: updateData },
      { new: true, runValidators: true }
    );

    if (!updatedCargo) {
      return res.status(404).json({ message: 'Cargo not found' });
    }

    cargoEventEmitter.emit('cargoUpdated', updatedCargo);

    return res.status(200).json({
      message: 'Cargo updated successfully',
      cargo: updatedCargo,
    });
  } catch (error) {
    console.error("Error updating cargo:", error);
    return res.status(500).json({ message: 'Error updating cargo', error: (error as Error).message });
  }
};

Event Processing in the Backend System 📡
1. Introduction to Event Emitters
In this backend system, we utilize Node.js's EventEmitter class to implement an event-driven architecture. The eventEmitter.ts file initializes an instance of EventEmitter that allows us to manage and process events, such as trade transactions and cargo updates, in real time.

2. EventEmitter Initialization
The eventEmitter.ts file is where the event emitter is initialized:

import EventEmitter from 'events';

const tradeEventEmitter = new EventEmitter();

export default tradeEventEmitter;

a. Ingesting Events
When a trade transaction is initiated, an event is emitted to signal that a specific action needs to be performed, such as updating cargo information or adjusting inventory levels.

b. Processing Events in Real-Time
The event emitter listens for these events and triggers associated callback functions that process the event data. This ensures that actions like updating the cargo status and inventory quantities happen immediately after a trade is initiated, maintaining consistency across the system.

4. Example: Handling a Trade Initiation Event
Below is an example of how tradeEventEmitter is used in conjunction with other components like the Cargo and Inventory models to process a trade event:

Same we have implemented for managing cargo where on updating some data on the cargo we call cargoEventEmittter.ts file.

import tradeEventEmitter from './eventEmitter';
import Cargo from '../model/cargo';
import Inventory from '../model/inventory';

tradeEventEmitter.on('tradeInitiated', async (trade) => {
    try {
      const newCargo = new Cargo({
        shipmentId: `shipment-${trade.transactionId}`,
        origin: trade.seller,
        destination: trade.buyer,
        items: trade.items,
        status: 'pending',
        ETA: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000) 
      });
  
      await newCargo.save();
  
      await Inventory.updateOne(
        { stationId: trade.buyer },
        {
          $inc: { quantity: trade.items.length },
          $addToSet: { items: { $each: trade.items } }
        }
      );
      
      // Update inventory for the seller
      await Inventory.updateOne(
        { stationId: trade.seller },
        {
          $inc: { quantity: -trade.items.length },
          $pullAll: { items: trade.items }
        }
      );
      console.log(`Cargo created and inventory updated for trade ${trade.transactionId}`);
    } catch (error) {
      console.error("Error processing trade event:", error);
    }
});

Understanding updates.controllers.ts 📡
1. Purpose of updates.controllers.ts
The updates.controllers.ts file is responsible for managing real-time updates in the system. It leverages Server-Sent Events (SSE) to push real-time data from the server to the client whenever specific events occur, such as a trade initiation or a cargo update. This functionality is crucial for maintaining a live feed of updates, ensuring that users have the latest information without needing to constantly refresh the page.

2. Key Components of the File
a. Setting Up Server-Sent Events (SSE)
The realTimeUpdateDetailFunction sets up the connection for SSE. This allows the server to push updates directly to the client.

res.setHeader('Content-Type', 'text/event-stream');
res.setHeader('Cache-Control', 'no-cache');
res.setHeader('Connection', 'keep-alive');

// Send a message to confirm the connection is established
res.write('data: Connection established\n\n');

Access this endpoint via route

 domain/api/updates/real-time/

Importance: This mechanism ensures users receive immediate updates without needing to refresh the page, maintaining the integrity and responsiveness of the system.
🔍 Knowning Limitations and Potential Improvements :
Known Limitations ->
Scalability: The current implementation may encounter issues as the number of trades and cargo updates increases, especially with real-time features.
Single Database: Using a single MongoDB instance could create bottlenecks. Sharding or replication might be needed for large-scale deployments.
Secruity Vulnerabilities : The system currently lacks advanced security measures, leaving it vulnerable to potential threats like unauthorized access and data breaches.
Limited Real-Time Analytics : No real-time analytics for monitoring.
Potential Improvements ->
Caching: As trade and cargo volume increases, consider sharding or replication for MongoDB.
Microservices Architecture:Transitioning to a microservices architecture could enhance scalability and maintainability.
Robust Security : Implement robust security protocols, including encryption, multi-factor authentication, and regular security audits to safeguard the system.


