This is AI chess game made from python with the help of pygames and math and sys libraires. In this project i have used maximum and minimum algorithm to make it functional like a human.

        This is time complexity formy project.
Minimax Algorithm (AI Decision):
Without Alpha-Beta Pruning:
O(b^d)

b = branching factor (~30 average moves per position)
d = search depth (3 in your code)
Result: O(30^3) ≈ O(27,000) operations
With Alpha-Beta Pruning (your code uses this):
O(b^d/2)

Result: O(30^{1.5}) ≈ O(164) operations – roughly 150× faster
Overall Game Loop:
Each frame: Draw operations O(64) + AI move with minimax at depth 3
Dominant factor: Minimax = O(b^(d/2)) per AI turn
