# PoW fraud proofs and Light clients

Davidson | 2023-10-04 15:18:49 UTC | #1

I'm writing a small [blog post](https://blog.dlsouza.lol/2023/09/28/pow-fraud-proof.html) about the applicability of PoW fraud proofs towards improving light client security, while retaining the low resource usage of such clients. The post summarizes my experiments with a lightweight node implementation that I'm working on, [Floresta](https://github.com/Davidson-Souza/floresta), and how we would ship this idea into clients that work on contained devices, like smartphones.

The idea is still very early on, and I would love to see some feedbacks on the idea and general design. There are some open problems, specially related to DoS, any solution to this would also be much appreciated.

-------------------------

