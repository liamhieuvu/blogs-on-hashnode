## Easy path from traditional to blockchain dev

The whole world pays lots of attention to blockchain recently. If you are frontend or backend developers, how to work in the blockchain field without starting like a fresher? I made a change from traditional systems to the blockchain world in a hard way, but you don't need to do that path. I write this blog to share an easier way for anyone who needs it.

# Where I am - Architecture in traditional backend view

We usually work on systems where business logic is implemented on the backend side. These systems, e.g., E-Commerce, FinTech, and EdTech store and retrieve data in databases. Then, the backend side conduct some kinds of logic, e.g., checking participating rules, inventory status, identity verification, etc. Finally, frontend makes up beautiful UI for users.

![Screen Shot 2022-04-05 at 12.29.25 AM-min.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1649093771541/tyQyB7Fgc.png)

The point is that everything is centralized! Someone in charge can still manipulate the systems. For example, they can change the participating rules in an online course or edit the purchase history. It's alright because the business requires that, and the users must trust the system. And we, as backend engineers, know the importance of data consistency and system reliability. A tiny mistake can cost a lot of money. However, is it true in the blockchain world?

# Where I will be - Architecture in blockchain view

For the backend side, the answer is no and ... yes but not strict. "No" if the system is fully decentralized, and "yes" if it is half decentralized. However, the frontend side has more complex work to do. It may interact with "2" backend systems, the first one is the blockchain network, and the second one is the traditional backend. So, what does "decentralized" mean? I tried to Google search, and my mind got a mess. Make it simple: anyone can hold data and check logic, not just an authority. Because of that, there are different people and organizations running nodes around the world. These nodes run the same software, hold the same data, and can verify business logic like payment transactions.


- Backend as data aggregator:

![Screen Shot 2022-04-05 at 12.39.49 AM-min.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1649094051310/joJoo0UO6.png)

- Backend as data feeder:

![Screen Shot 2022-04-05 at 12.35.50 AM-min.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1649093869070/EI8NNqP6A.png)

# Things to begin with

After knowing the big picture, let's get our hands dirty. The first thing I suggest is to learn about how a blockchain run in general. Remember: `general` not detail! The best source I found is Blockchain 101. Thanks Anders Brownworth. 100k likes to you.

- https://www.youtube.com/watch?v=_160oMzblY8
- https://www.youtube.com/watch?v=xIDL_akeras


![Screen Shot 2022-04-05 at 12.47.02 AM-min.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1649094446587/1Grg7tw1p.png)

The keys to learn from the above videos:

- SHA256 Hash of data
- What is a block? What constructs a block?
- How to chain a block?
- How to mine a block?
- Who holds the chain a block, aka blockchain?
- What are transactions? How many of them can be packed into a block?
- Public and private key pairs
- What is the signature of some kinds of data?
- How to construct a transaction with private keys?

After mastering these terms, you are good to go. If you are a frontend developer, you need to learn how to interact with dApps, aka smart contracts, to issue a business request. The rest are making up beautiful UI, what you are good at. If you are a backend developer, you need to learn how to crawl on-chain data through view functions of smart contracts or subgraphs.

# More things to learn

You may work on a specific domain in dApps, e.g. DeFi, NFT. Or you even work on layer 1 of a blockchain (the node software). You can find more knowledge and path here:

- https://github.com/OffcierCia/DeFi-Developer-Road-Map
- https://github.com/Envoy-VC/blockchain-app-developer-roadmap


Thank you for reading my blog!