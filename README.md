# CHESS-ALML-game-by-python-maximum-and-minimum-algorithm.
This is AI Chess game made by python, pygame, sys, math libraries . This game is on concept of maximum and minimum algorithm.
Minimax Algorithm (AI Decision):
Without Alpha-Beta Pruning:

b = branching factor (~30 average moves per position)
d = search depth (3 in your code)
Result: O(30^3) ≈ O(27,000) operations
O(b^d)

With Alpha-Beta Pruning (your code uses this): O(b^d/2)
Result: O(30^{1.5}) ≈ O(164) operations – roughly 150× faster
Overall Game Loop:
Each frame: Draw operations O(64) + AI move with minimax at depth 3
Dominant factor: Minimax = O(b^(d/2)) per AI turn
Summary:
