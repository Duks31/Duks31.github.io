---
title: 'Monolith By TikTok (ByteDance)'
date: 2025-02-19
permalink: /posts/2025/02/Monolith By TikTok ByteDance/
tags:
  - Artificial Intelligence
  - Data Science
  - Machine Learning
---

Social media timelines are so captivating because they’re powered by incredibly sophisticated recommendation systems that seem almost magical in how well they understand user preferences. TikTok is a master of this, delivering a personalized feed that keeps users hooked for hours. The secret behind this is ByteDance’s cutting-edge recommendation engine, [Monolith](https://arxiv.org/pdf/2209.07663) — a scalable, real-time system designed to optimize engagement like never before.

Recommendation systems can be built using production-scale deep learning frameworks like Pytorch and Tensorflow but they fall short of reaching the main business demands. In making Monolith, they came up with a **collision-less embedding** table with clever optimizations like expiring embeddings and frequency filtering and also **production-ready online training** architecture with high fault-tolerance, then finally they also proved that **system reliability could be a trade-off** for real-time learning.

The common goal of a business is to deliver personalized content for each individual user in real time and key challenges arise when building recommendation systems

* The features are mostly sparse, categorical and dynamically changing causing collision

* The distribution of training data is non-stationary, i.e. the user may have change of content preferences overtime

* The system should be scalable i.e. memory efficiency and fault-tolerant designs

This is where monolith comes in. It was designed to address the above challenges:

* A collision-less embedding using Cuckoo HashMap to prevent data collisions.

* Real-time Online Training, allowing the system to learn from user interactions instantly.

* Efficient synchronization to ensure that the model does not have any quality loss or computation overhead.

### Collision-less Embedding

In recommendation systems, an **embedding** is a way to represent things (like users, videos, or hashtags) as numbers so that computers can understand and compare them. Think of it like converting words into points on a map, where similar things are closer together.

Imagine you have two different users, but they accidentally get placed on the exact same spot on the map. In a recommender system, this means the system thinks they are the same person with identical tastes — bad news for personalized recommendations! This happens because of **collisions** when two different IDs (e.g., user IDs or video IDs) are mapped to the same embedding by accident.

![Cuckoo HashMap](https://cdn-images-1.medium.com/max/2000/0*TPDjc6YQuxd1CILN.png)

Cuckoo HashMap

Monolith’s solution to this is using the **Cuckoo HashMap**, it is like having two maps for each ID. If an ID doesn’t fit on the first map, it gets moved to its spot on the second map — just like a cuckoo bird kicks out eggs from a nest to make space for its own.

How it works:

**Two Hash Functions:**

* Monolith uses two different hash functions (h0 and h1) to calculate two possible spots for each ID.

* This gives each ID two chances to find a spot, reducing the chance of a collision.

**Placing Feature Embeddings:**

* When storing an embedding, Monolith first checks the spot given by h0.

* If that spot is taken, the existing item is **kicked out** and moved to its other spot, calculated by h1.

* This process repeats until all items find a home.

**Collision Avoidance:**

* By allowing each ID to have two potential spots, Monolith drastically reduces the chances of collisions.

* This ensures each feature ID gets its own unique embedding, preserving the integrity of the data.

To explain it better look at this scenario:

Imagine you have two lockers at school. You always check the first one (h0) to store your backpack. If someone else’s bag is already there, you move it to their second locker (h1) and take the first one. If that locker is also full, the shuffling continues until all bags have a place. This system ensures no two bags end up in the same locker, just like Monolith ensures no two IDs share the same embedding.

![Monolith Online Training Architecture](https://cdn-images-1.medium.com/max/2556/0*gItQNzJTaVAbBnoH.png)

Monolith Online Training Architecture

### Online Training

The Monolith training is divided into two stages Batch Training stage and Online Training stage

The **batch training** is similar to traditional training processes it involves:

**Data Preparation**: Historical batch data is collected and stored in a system like HDFS (Hadoop Distributed File System). This data typically includes user interactions (e.g., clicks, views) and associated features (e.g., user IDs, item IDs).

**Training Worker**: A training worker reads a mini-batch of this historical data. The worker then performs the following steps:

* **Embedding Table Lookup**: The worker looks up the embeddings for the feature IDs (e.g., user IDs, item IDs) in the embedding table stored on the **Training Parameter Server (PS)**. These embeddings are dense vector representations of the sparse categorical features.

* **Model Forward and Backward Pass**: The worker performs a forward pass to make predictions using the model (e.g., a DeepFM model) and a backward pass to compute gradients based on the prediction errors.

* **Gradient Updates**: The worker sends the computed gradients back to the Training PS, which updates the model parameters (both dense weights and sparse embeddings).

**Parameter Server (PS)**: The Training PS stores the model parameters, including the embedding table. During batch training, it receives gradient updates from the training worker and applies them to the model parameters. This process is repeated for multiple epochs until the model converges.

The batch training stage is essential for training the model on a large volume of historical data, ensuring that it learns general patterns and user preferences. However, since this stage relies on static data, it cannot adapt to real-time user feedback.

Once the model is deployed for online serving, the **online training stage** takes over. This stage is designed to adapt the model in real-time based on the latest user interactions. Here’s how it works:

**Real-Time Data Streaming**: Instead of using historical data, the system now consumes real-time data streams. These streams include user actions (e.g., clicks, likes) and associated features, which are logged in real-time using a system like Kafka.

**Training Worker**: Similar to batch training, the training worker processes the real-time data:

* **Embedding Table Lookup**: The worker looks up the embeddings for the feature IDs in the embedding table stored on the Training PS.

* **Model Forward and Backward Pass**: The worker performs a forward pass to make predictions and a backward pass to compute gradients based on the latest user feedback.

* **Gradient Updates**: The worker sends the gradients to the Training PS, which updates the model parameters.

**Parameter Synchronization**: One of the key features of Monolith is its ability to synchronize parameters between the **Training PS** and the **Serving PS** in near real-time. This is done at regular intervals (e.g., every minute) to ensure that the online serving model reflects the latest updates. Only the parameters that have changed (e.g., embeddings for recently active users or items) are synchronized, reducing the computational and network overhead.

**Serving PS and Model Server**: The Serving PS stores the latest version of the model parameters, which are used by the **Model Server** to serve real-time recommendations to users. When a user makes a request (e.g., loading a feed), the Model Server performs a forward pass using the latest parameters to generate personalized recommendations.

**User Actions**: The user interacts with the recommendations (e.g., clicking on an item), and these actions are logged and fed back into the system as real-time data, creating a continuous feedback loop.

![Streaming Engine](https://cdn-images-1.medium.com/max/2912/0*OP278VF08Szx0ALS.png)

The **streaming engine** is the core infrastructure that enables Monolith to handle both batch and online training seamlessly. It acts as the bridge between historical data (used in batch training) and real-time data (used in online training). Here’s how it fits into the overall process:

**Data Ingestion**: The streaming engine uses **Kafka queues** to log two types of data:

* **User Actions**: Real-time logs of user interactions (e.g., clicks, likes).

* **Features**: Metadata associated with users and items (e.g., user IDs, item IDs, categories).

**Data Processing**: A **Flink streaming job** processes these logs in real-time. It performs the following tasks:

* **Joining Data**: The streaming engine combines user actions with their corresponding features to create training examples. This is where the **online joiner** comes into play (more on this below).

* **Writing to Queues**: The joined training examples are written to a Kafka queue, which serves as the input for both batch and online training.

**Flexibility**: The streaming engine allows Monolith to switch between batch and online training modes. For batch training, the data is dumped to HDFS and processed in large chunks. For online training, the data is consumed directly from the Kafka queue in real-time.

![Online Joiner](https://cdn-images-1.medium.com/max/2912/0*rz_aF1dY4BLmRIxx.png)

The **online joiner** is a critical component of the streaming engine, responsible for pairing user actions with their corresponding features in real-time. This is especially important because user actions and features may arrive out of order or with delays. Here’s how it works:

**Unique Key Matching**: Each user action and feature is assigned a unique key (e.g., a request ID). The online joiner uses these keys to correctly pair actions with their corresponding features.

**Handling Delays**: In real-world scenarios, there can be delays between when a user sees an item and when they interact with it (e.g., a user might click on an item days after seeing it). To handle this:

* **In-Memory Cache**: The online joiner first checks an in-memory cache for features that match the user action.

* **On-Disk Storage**: If the feature is not found in the cache, the joiner looks it up in an on-disk key-value storage. This ensures that even delayed actions can be paired with their features.

**Output**: The joined data (user actions + features) is written to a Kafka queue, which is then consumed by the training workers for online training.

Once the model is updated during online training, the latest parameters need to be synchronized with the serving system to ensure real-time recommendations. This is where **parameter synchronization** comes into play:

**Incremental Updates**: Instead of transferring the entire model (which can be several terabytes in size), Monolith only synchronizes the subset of parameters that have been updated since the last synchronization. This is done by maintaining a **hash set of touched keys** (IDs whose embeddings have been updated).

**Frequency of Updates**:

* **Sparse Parameters**: These are updated frequently (e.g., every minute) because they represent user and item embeddings, which change rapidly based on real-time interactions.

* **Dense Parameters**: These are updated less frequently (e.g., daily) because they represent slower-changing weights in the neural network.

**Efficiency**: By synchronizing only the updated parameters, Monolith minimizes network bandwidth and memory usage, ensuring that the system can scale to handle large-scale production workloads.

**Fault Tolerance**: To handle potential failures in the Parameter Servers (PS), Monolith periodically snapshots the state of the training PS. In case of a failure, the system can recover from the latest snapshot, ensuring minimal disruption to the training process.

### Fault Tolerance

Monolith ensures system reliability through **snapshotting**, which periodically saves the state of the model parameters stored on the **Parameter Servers (PS)**. Here’s how it works:

**Daily Snapshots**:

* Monolith takes snapshots of the training PS once a day, storing the current state of all model parameters (dense weights and sparse embeddings).

* If a PS fails, the system recovers from the latest snapshot, losing at most one day’s worth of updates.

**Trade-offs**:

* **Model Quality**: Daily snapshots minimize the impact of failures, as only a small subset of embeddings (sparse parameters) is updated in a short time window. Dense parameters change slowly, so losing a day’s updates has little effect.

* **Computation Overhead**: Daily snapshots reduce the strain on the system compared to more frequent snapshots, which would require more memory and disk I/O.

**Why It Works**:

* **Sparse Updates**: Only a small fraction of embeddings are updated daily, so losing a day’s updates affects very few users.

* **User Distribution**: With user IDs evenly distributed across PS machines, a failure impacts only a tiny fraction of daily active users (e.g., 0.01%).

**In Practice**:

* Monolith’s fault tolerance has been tested in production, handling PS failures (e.g., 0.01% failure rate) with minimal impact on model quality or user experience.

* The system recovers quickly, ensuring high availability and reliability.

And that’s about it, machine learning real world use cases are always fun to encounter and understand. You can read the full paper [here](https://arxiv.org/pdf/2209.07663).

*Images are from the [paper](https://arxiv.org/pdf/2209.07663).*

Stay Nerdy…

If you liked this, you would like more [here](https://ncep.substack.com/)
