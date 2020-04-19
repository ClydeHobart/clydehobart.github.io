---
layout: page
title: Code Samples
tagline: Some examples of code I've written for various projects
permalink: /code-samples
ref: code-samples
order: 0
---

For more details on individual projects, check out the posts on my [homepage]({{ '/' | absolute_url }}), or check out my [GitHub]({{ site.github.owner_url }}).

## Contents
* [`solutionGenerator.py`](#solutiongeneratorpy)
* [`OH_BOI_BasicChessAI.cs`](#ohboibasicchessaics)
* [Roguelike Procedural Generation Scripts](#roguelike-procedural-generation-scripts)
    * [`Cell.cs`](#cellcs)
    * [`Door.cs`](#doorcs)
    * [`Room.cs`](#roomcs)
    * [`RoomManager.cs`](#roommanagercs)
    * [`RPGUtil.cs`](#rpgutilcs)

## `solutionGenerator.py`

{% highlight py %}
import sys, math, pygame#, numpy, random
from collections import deque
Vector3 = pygame.math.Vector3

"""
Zeke Baker | 2020-01-18
This program is intended to fill out the state space for my favorite twisty puzzle,
the Pentultimate (http://www.twistypuzzles.com/cgi-bin/puzzle.cgi?pkey=1741). The
idea was to write a program that would assemble, through BFS, an "all roads lead to
Rome"-type graph to provide the shortest move list to solve the puzzle. It didn't
take long into testing for me to realize that not only was the time this would need
to fill the graph extremely large, but also the memory needed to store it was more
than I had really thought about and/or anticipated. So the problem still lies
unsolved, but the code up until the BFS (and the BFS process itself) is still solid.

The numbering system used to designate piece locations and default orientations can
be found here: http://bit.ly/3alMiRq . Note that for the triangular pieces, some
times they're indexed 0-19, and others they're indexed 12-31, depending on the need
"""

phi = (1 + math.sqrt(5)) / 2
n_pents = 12
n_tris = 20
n_pieces = n_pents + n_tris

# These were selected so that pentagon 0 stays stationary. Note that any turn of one
# pentagon is equivalent to the same turn about the opposite pentagon.
rotatable_pents = (2, 3, 5, 7, 10, 11)

# A tuple of 32 vector3s, for the normal vectors of the pieces.
# Triangular pieces have indices 12-31 here.
norm_vecs = None

# A tuple of 32 vector3s, for the directional vectors of the pieces.
# Triangular pieces have indices 12-31 here.
dir_vecs = None

# A dict of 64 {string: vector3} entries, since performing rotations on vector3s
# isn't always super accurate. Rotating a vector, rounding it to 3 places after
# the decimal, then checking for the corresponding resulting vector is accurate.
vec_dict = None

# A list of 24 lists of 32 lists of 2 ints. Outermost index indicates which
# transformation is applied, the middle index indicates which piece is affected,
# and the innermost index chooses between the locational index difference and the
# rotational difference. This'll be clarified in init_transforms().
transforms = None

def average_vecs(vecs):
    """Returns the unit vector of the average of the supplied vector3s.

    Parameter:
        vecs (sequence of vector3s): vector3s to be averaged

    Returns:
        vector3: unit vector of the average of vecs"""

    avg = Vector3(0, 0, 0)

    for vec in vecs:
        avg += vec

    return avg.normalize()

def round_vector3(vec3, n):
    """Rounds the vector3s to a specified number of digits after the decimal.

    Parameters:
        vec3 (vector3): vector3 to be rounded
        n    (int):     number of spaces after the decimal to round to

    Returns:
        vector3: copy of vec3, rounded to n digits after the decimal"""

    vec3 = Vector3(vec3)
    vec3.x = round(vec3.x, n)

    # This and the subsequent similar lines set negative 0 to positive 0
    if vec3.x == 0: vec3.x = 0

    vec3.y = round(vec3.y, n)

    if vec3.y == 0: vec3.y = 0

    vec3.z = round(vec3.z, n)

    if vec3.z == 0: vec3.z = 0

    return vec3

def init_vecs():
    """Initializes the norm_vecs, dir_vecs, and vec_dict variables."""

    global norm_vecs, dir_vecs, vec_dict
    vec_dict = dict()

    pent_norms = [None] * n_pents
    tri_norms = [None] * n_tris

    # The 12 pentagonal pieces and the first 12 triangular pieces have similar
    # normal vectors
    for piece in range(n_pents):
        perm = piece >> 2
        a = (1, -1)[(piece >> 1) % 2]
        b = (1, -1)[piece % 2]
        pent_temp = (a * phi, b, 0)
        tri_temp = (a * phi, 0, b / phi)
        pent_norms[piece] = Vector3(
            pent_temp[(0 + perm) % 3],
            pent_temp[(1 + perm) % 3],
            pent_temp[(2 + perm) % 3]).normalize()
        tri_norms[piece] = Vector3(
            tri_temp[(0 + perm) % 3],
            tri_temp[(1 + perm) % 3],
            tri_temp[(2 + perm) % 3]).normalize()

    # The last 8 triangular pieces have normal vectors corresponding to the
    # corners of a cube
    for tri in range(n_pents, n_tris):
        a = (-1, 1)[(tri >> 2) % 2]
        b = (1, -1)[(tri >> 1) % 2]
        c = (1, -1)[tri % 2]
        tri_norms[tri] = Vector3(a, c, b).normalize()

    pent_dirs = [None] * n_pents
    tri_dirs = [None] * n_tris

    # The directional vector of a piece points towards one of the adjacent,
    # similarly-shaped pieces, but is still perpendicular to the piece's
    # normal vector. The "... = norm_1.rotate..." lines accomplish this
    for piece in range(n_tris):
        if piece < n_pents:
            # If the piece's index is < 12, the directional vector points towards next
            # piece for even pieces, or the previous piece for odd pieces.
            norm_1 = pent_norms[piece]
            norm_2 = pent_norms[piece + (1, -1)[piece % 2]]
            pent_dirs[piece] = norm_1.rotate(90, norm_1.cross(norm_2 - norm_1)).normalize()
            norm_1 = tri_norms[piece]
            norm_2 = tri_norms[piece + (1, -1)[piece % 2]]
            tri_dirs[piece] = norm_1.rotate(90, norm_1.cross(norm_2 - norm_1)).normalize()
        else:
            norm_1 = tri_norms[piece]
            norm_2 = tri_norms[(piece - n_pents) >> 1]
            tri_dirs[piece] = norm_1.rotate(90, norm_1.cross(norm_2 - norm_1)).normalize()

    norm_vecs = tuple(pent_norms + tri_norms)
    dir_vecs = tuple(pent_dirs + tri_dirs)

    for norm_vec in norm_vecs:
        vec_dict.update({str(round_vector3(norm_vec, 3)): norm_vec})

    for dir_vec in dir_vecs:
        vec_dict.update({str(round_vector3(dir_vec, 3)): dir_vec})

def affected_pieces(piece):
    """Returns a list of the indices of all pieces affected by twisting the puzzle
    about a piece. Assumes init_vecs() has been called.
    
    Parameter:
        piece (int): index of the pentagonal piece to twist the puzzle about
    
    Returns:
        List: The indices of all pieces (pentagonal and triangular [12-31 again])
              affected by twisting the puzzle about piece"""
    pieces = []

    for old_piece in range(len(norm_vecs)):
        if norm_vecs[old_piece].angle_to(norm_vecs[piece]) < 90:
            pieces.append(old_piece)

    return pieces

def dir_diff(new_piece, old_dir, new_dir):
    """Returns the "difference" between the two directional vectors for a given
    piece. Assumes init_vecs() has been called.
    
    Parameters:
        new_piece (int): index of the new normal vector for a piece after being
            twisted
        old_dir (vector3): directional vector of the piece after being twisted
        new_dir (vector3): default directional vector of a piece in that
            location
    
    Returns:
        int: in [0, 5) for a pentagonal piece or [0, 3) for a triangular piece.
            Indicates the number of turns (assuming right-hand rule twists) to
            be applied from the default vector (new_dir) to get to the present
            vector (old_dir)"""
    # An example of what this could return (refer to the image linked in the
    # first docstring): starting with a solved puzzle, perform 1 twist about
    # piece 2. Piece 8 is now in the location 9 (the original location of piece
    # 9), and the directional vector for location 9 needs to be twisted 3 times
    # about location 9's normal vector to arrive to where piece 8's directional
    # vector currently points. I know, this is all probably very confusing and
    # verbose, but it provides a distinction between states where pieces are in
    # the same locations but not the same orientations.

    angle = None

    if new_dir == old_dir:
        angle = 0
    else:
        angle = old_dir.angle_to(new_dir)

    angle = round(angle)

    if angle != 0:
        if norm_vecs[new_piece].dot(old_dir.cross(new_dir).normalize()) < 0:
            angle = 360 - angle
        angle = 360 - angle

    return (angle // 120, angle // 72)[new_piece < n_pents]

def init_transforms():
    """Initializes the transforms variable. Assumes init_vecs() has been called."""

    global transforms

    transforms = [None] * 24

    for t in range(len(transforms)):
        transforms[t] = [[0, 0] for _ in range(n_pieces)]
        piece = rotatable_pents[t >> 2] # The piece being twisted
        
        # Rotations (or twists). For a pentagonal piece, which all rotatable
        # pieces are, 1 "rotation" is 72 degrees, or 1 fifth of a "rotation" in
        # the traditional sense of the word.
        rots = 1 + t % 4 
        pieces = affected_pieces(piece)

        for old_piece in pieces:
            new_piece = norm_vecs.index(vec_dict[str(round_vector3(
                norm_vecs[old_piece].rotate(72 * rots, norm_vecs[piece]), 3))])

            # With 0 as the innermost index, it indicates the index offset to
            # get back to index old_piece
            transforms[t][new_piece][0] = (old_piece - new_piece) % n_pieces
            diff = dir_diff(new_piece,
                dir_vecs[old_piece].rotate(72 * rots, norm_vecs[piece]),
                dir_vecs[new_piece])

            # With 1 as the innermost index, it indicates the directional
            # difference returned by dir_diff(). Note that this should already
            # be in the proper range, but the "diff % n" just re-ensures it
            transforms[t][new_piece][1] = diff % (3, 5)[new_piece < n_pents]

def state_to_bytes(s):
    """Converts a state to a bytes object.
    
    Parameter:
        s (list): list of 32 lists of 2 ints. First index is the location. For
        the second index, 0 provides the piece at that location, and 1 provides
        the directional difference (as provided by dir_diff())
    
    Return:
        bytes: A compressed bytes representation of the state. The bitstring
            equivalent is of the following form:
        bbbb bbb bbbb bbb ... bbbb bbb bbbb bbb   bbbbb bb bbbbb bb ... bbbbb bb bbbbb bb
        ^    ^   ^    ^       ^    ^   ^    ^     ^     ^  ^     ^      ^     ^  ^     ^
        |    |   |    |       |    |   |    |     |     |  |     |      |     |  |     |
        s[0][0]  s[1][0]      s[10][0] s[11][0]   s[12][0] s[13][0]     s[30][0] s[31][0]
             |        |            |        |           |        |            |        |
             s[0][1]  s[1][1]      s[10][1] s[11][1]    s[12][1] s[13][1]     s[30][1] s[31][1]"""

    # The easiest way that I could determine to compress this into bytes was
    # with a hex string
    hex_str = ""

    # First compress the state data for the pentagonal pieces.
    # For m in [0, 12), s[m][0] gives a number n in [0, 12), which requires 4 bits
    # For m in [0, 12), s[m][1] gives a number n in [0, 5), which requires 3 bits
    # Since each pentagonal piece requires 4 + 3 = 7 bits, 4 pieces are required
    # to write a hex string with all bits being utilized
    for chunk in range(0, n_pents, 4):
        val = 0

        for pent in range(chunk, chunk + 4):
            val <<= 4
            val |= s[pent][0]
            val <<= 3
            val |= s[pent][1]

        # 7 bits * 4 * (1 hex digit / 4 bits) = 7 hex digits
        hex_str += "{:07x}".format(val)

    # Next, compress the state data for the triangular pieces.
    # For m in [12, 31), s[m][0] gives a number n in [12, 31), which requires 5 bits
    # Note that even if 12 were subtracted from each value, 5 bits would still be needed
    # For m in [12, 31), s[m][1] gives a number n in [0, 3), which requires 2 bits
    # Since each triangular piece requires 5 + 2 = 7 bits, 4 pieces are required
    # to write a hex string with all bits being utilized
    for chunk in range(n_pents, n_pieces, 4):
        val = 0

        for tri in range(chunk, chunk + 4):
            val <<= 5
            val |= s[tri][0]
            val <<= 2
            val |= s[tri][1]

        # 7 bits * 4 * (1 hex digit / 4 bits) = 7 hex digits
        hex_str += "{:07x}".format(val)

    return bytes.fromhex(hex_str)

def bytes_to_state(b):
    """Reconverts a bytes object to a state.
    
    Parameter:
        b (bytes): A bytes object representation of a state. Follows the same
            description defined in state_to_bytes(), which it should be the
            return value of.
    
    Returns:
        list: list of 32 lists of 2 ints. Follows the same description of a
            state defined in state_to_bytes()."""

    state = [[0, 0] for _ in range(n_pieces)]
    hex_str = b.hex()

    # For decompressing into a state, still break the bytes off in 7 hexadecimal
    # digit chunks
    for chunk in range(0, n_pents, 4):
        val = int(hex_str[:7], 16)
        hex_str = hex_str[7:]

        # Unpack the chunks in reversed order so that we can easily address the bits
        for pent in reversed(range(chunk, chunk + 4)):
            state[pent][1] = val & 0b0111
            val >>= 3
            state[pent][0] = val & 0b1111
            val >>= 4

    # For decompressing into a state, still break the bytes off in 7 hexadecimal
    # digit chunks
    for chunk in range(n_pents, n_pieces, 4):
        val = int(hex_str[:7], 16)
        hex_str = hex_str[7:]

        # Unpack the chunks in reversed order so that we can easily address the bits
        for tri in reversed(range(chunk, chunk + 4)):
            state[tri][1] = val & 0b00011
            val >>= 2
            state[tri][0] = val & 0b11111
            val >>= 5
    
    return state

def apply_transform(state_as_bytes, transform):
    """Applies a transformation to a state that's in bytes form. Assumes
    init_transforms() has been called.
    
    Parameters:
        state_as_bytes (bytes): The state to be transformed, represented as a 
            bytes object. Should be in the same format as the output from
            state_to_bytes().
        transform (int): Index of transformation in transforms list to apply.
        
    Returns:
        bytes: The transformed state as a bytes object, in the format defined
            by state_to_bytes()."""

    old_state = bytes_to_state(state_as_bytes)
    new_state = [None] * n_pieces

    for new_loc in range(n_pieces):
        # First find the index this piece was previously at
        old_loc = (new_loc + transform[new_loc][0]) % n_pieces
        # The piece at this location is the same as the piece at the old location,
        # and the direction must be readjusted
        new_state[new_loc] = [old_state[old_loc][0], (old_state[old_loc][1] + transform[new_loc][1]) % (3, 5)[new_loc < n_pents]]
    
    return state_to_bytes(new_state)

def inv_transform(t):
    """Returns the index of the inverse transformation.
    
    Parameter:
        t (int): Index of the transformation to be inverted
    
    Returns:
        int: Index of the inverse transformation of t."""
    
    # This is just a more compact form of "(t - (t % 4)) + (3 - (t % 4))"
    # The transformations are defined in groups of 4, where each group all
    # rotate the same piece, just different amounts of twists. For each group,
    # the twist amounts follow the sequence (1, 2, 3, 4). Since the pieces being
    # twisted have 5-way rotational symmetry, a transformation of 1 twist is
    # undone by a transformation of the same piece of 4 twists, and vice versa.
    # The same is true for 2 and 3 twists (2 -> 3 and 3 -> 2).
    return t + 3 - 2 * (t % 4)

def bfs(limit_depth=False, max_depth=3):
    """Performs Breadth-First Search on the state space, starting with the puzzle
    in the solved state. Outputs updates whenever the depth changes, if the state
    count is less than 1000, or if all digits after the first 3 are 0. Updates are
    printed to the same line to prevent superfluous output. Assumes
    init_transforms() has been called.
    
    Parameters:
        limit_depth (bool): If true, the search will stop at depth max_depth.
        max_depth (int): Maximum depth of the search, only used when limit_depth
            == true"""

    global state_tree
    initial_node = state_to_bytes([[piece, 0] for piece in range(n_pieces)])
    frontier = deque()
    frontier.append(initial_node)

    # state_tree is a dict representing the state tree of the search, with nodes
    # mapping to a tuple of their depth, the transformation to get to their
    # parent node, and the parent node. Nodes are in bytes form.
    state_tree = {initial_node: [0, None, None]}
    states = 0
    depth = 0

    while True:
        if len(frontier) == 0:
            break

        node = frontier.popleft()
        states += 1
        new_depth, _, _ = state_tree[node]
        log_10_states = int(math.log10(states))

        if new_depth != depth or log_10_states < 3 or states % 10 ** (log_10_states - 2) == 0:
            print("depth: {: >3d}, states: {: >16d}".format(new_depth, states), end="\r")

        depth = new_depth

        if limit_depth and depth == max_depth:
            continue

        for t in range(len(transforms)):
            child = apply_transform(node, transforms[t])

            if child not in state_tree:
                state_tree.update({child: [depth + 1, inv_transform(t), node]})
                frontier.append(child)

def norms_and_dirs():
    """Returns a formatted string displaying the normal vectors and directional
    vectors. Assumes init_vecs() has been called.
    
    Returns:
        string: Formatted versions of norm_vecs and dir_vecs. Note that
            triangular pieces are displayed as being indexed 0-19."""

    vecs_str = "norm_vecs:"

    for piece in range(n_pieces):
        if piece == 0:
            vecs_str += "\n\tpents:"
        elif piece == n_pents:
            vecs_str += "\n\ttris"

        if piece < n_pents:
            vecs_str += "\n\t\t{:02d}: {}".format(piece, str(norm_vecs[piece]))
        else:
            vecs_str += "\n\t\t{:02d}: {}".format(piece - n_pents, str(norm_vecs[piece]))

    vecs_str += "\ndir_vecs:"

    for piece in range(n_pieces):
        if piece == 0:
            vecs_str += "\n\tpents:"
        elif piece == n_pents:
            vecs_str += "\n\ttris"

        if piece < n_pents:
            vecs_str += "\n\t\t{:02d}: {}".format(piece, str(dir_vecs[piece]))
        else:
            vecs_str += "\n\t\t{:02d}: {}".format(piece - n_pents, str(dir_vecs[piece]))

    return vecs_str

def state_to_str(state):
    """Returns the state represented as a formatted string. Visually it looks
    like a two-row table, with left to right indicating ascending first index,
    the first row is for state[n][0] values, and the second row is for state[n][1]
    values.

    Returns:
        string: A readable presentation of the state, in the format described above.
            Note that there's a column of pipes to provide distinction between the
            pentagonal pieces and the triangular pieces."""

    state_str = ""

    for p in range(n_pieces):
        state_str += "{separator_pipe}{space}{index_offset: >+3d}".format(separator_pipe=("", " |")[p == n_pents], space=("", " ")[p != 0], index_offset=state[p][0])

    state_str += "\n"

    for p in range(n_pieces):
        state_str += "{separator_pipe}{space} +{orientation_offset}".format(separator_pipe=("", " |")[p == n_pents], space=("", " ")[p != 0], orientation_offset=state[p][1])

    return state_str

def transforms_to_str():
    """Assembles a formatted string of all the transforms. Assumes init_transforms()
    has been called.

    Returns:
        string: Formatted string of all the transforms, with descriptions of
            what each transform is doing to the puzzle."""

    transforms_str = ""

    for t in range(len(transforms)):
        piece = rotatable_pents[int(t / 4)]
        rots = 1 + t % 4
        transforms_str += "rotate {} turns about {:02d}:\n    ".format(rots, piece)
        transforms_str += state_to_str(transforms[t]).replace("\n", "\n    ")
        transforms_str += "\n"

    return transforms_str

if __name__== "__main__":
    init_vecs()
    print(norms_and_dirs())
    init_transforms()
    print(transforms_to_str())
    bfs()
{% endhighlight %}

[(to top)](#top)

## `OH_BOI_BasicChessAI.cs`

{% highlight cs %}
using System.Collections;
using System.Collections.Generic;
using System.Collections.ObjectModel;
using UnityEngine;
using ChessDotNet;

/*
 * Zeke Baker | 2020-01-18
 * This is a modified version of the code found at https://jsfiddle.net/q76uzxwe/1/ ,
 * first translated into C#, then with a heavy overhaul to make the Minimax function
 * iterative instead of recursive. Unity (which this was implemented in) doesn't
 * have any means of threading that I could find, so this chess AI needed to be implemented
 * using a Unity Coroutine, which runs between frames.
 */

public class OH_BOI_BasicChessAI : MonoBehaviour
{
    private static bool DEBUG = false;
    private static int MIN_FPS = 15;
    private static float MAX_WAIT = 1 / MIN_FPS;
    private static int MAX_DEPTH = 2;
    private OH_BOI_ChessManager _chessManager;
    private int _passCount = 0;
    private int _passesSinceYield = 0;

    private static float[,] _pawnEvalWhite = {
        {0.0f,  0.0f,  0.0f,  0.0f,  0.0f,  0.0f,  0.0f,  0.0f},
        {5.0f,  5.0f,  5.0f,  5.0f,  5.0f,  5.0f,  5.0f,  5.0f},
        {1.0f,  1.0f,  2.0f,  3.0f,  3.0f,  2.0f,  1.0f,  1.0f},
        {0.5f,  0.5f,  1.0f,  2.5f,  2.5f,  1.0f,  0.5f,  0.5f},
        {0.0f,  0.0f,  0.0f,  2.0f,  2.0f,  0.0f,  0.0f,  0.0f},
        {0.5f, -0.5f, -1.0f,  0.0f,  0.0f, -1.0f, -0.5f,  0.5f},
        {0.5f,  1.0f,  1.0f, -2.0f, -2.0f,  1.0f,  1.0f,  0.5f},
        {0.0f,  0.0f,  0.0f,  0.0f,  0.0f,  0.0f,  0.0f,  0.0f}
    };

    private static float[,] _pawnEvalBlack = ReverseArray(_pawnEvalWhite);

    private static float[,] _knightEval = {
        {-5.0f, -4.0f, -3.0f, -3.0f, -3.0f, -3.0f, -4.0f, -5.0f},
        {-4.0f, -2.0f,  0.0f,  0.0f,  0.0f,  0.0f, -2.0f, -4.0f},
        {-3.0f,  0.0f,  1.0f,  1.5f,  1.5f,  1.0f,  0.0f, -3.0f},
        {-3.0f,  0.5f,  1.5f,  2.0f,  2.0f,  1.5f,  0.5f, -3.0f},
        {-3.0f,  0.0f,  1.5f,  2.0f,  2.0f,  1.5f,  0.0f, -3.0f},
        {-3.0f,  0.5f,  1.0f,  1.5f,  1.5f,  1.0f,  0.5f, -3.0f},
        {-4.0f, -2.0f,  0.0f,  0.5f,  0.5f,  0.0f, -2.0f, -4.0f},
        {-5.0f, -4.0f, -3.0f, -3.0f, -3.0f, -3.0f, -4.0f, -5.0f}
    };

    private static float[,] _bishopEvalWhite = new float[8, 8] {
        { -2.0f, -1.0f, -1.0f, -1.0f, -1.0f, -1.0f, -1.0f, -2.0f},
        { -1.0f,  0.0f,  0.0f,  0.0f,  0.0f,  0.0f,  0.0f, -1.0f},
        { -1.0f,  0.0f,  0.5f,  1.0f,  1.0f,  0.5f,  0.0f, -1.0f},
        { -1.0f,  0.5f,  0.5f,  1.0f,  1.0f,  0.5f,  0.5f, -1.0f},
        { -1.0f,  0.0f,  1.0f,  1.0f,  1.0f,  1.0f,  0.0f, -1.0f},
        { -1.0f,  1.0f,  1.0f,  1.0f,  1.0f,  1.0f,  1.0f, -1.0f},
        { -1.0f,  0.5f,  0.0f,  0.0f,  0.0f,  0.0f,  0.5f, -1.0f},
        { -2.0f, -1.0f, -1.0f, -1.0f, -1.0f, -1.0f, -1.0f, -2.0f}
    };

    private static float[,] _bishopEvalBlack = ReverseArray(_bishopEvalWhite);

    private static float[,] _rookEvalWhite = new float[8, 8] {
        {  0.0f,  0.0f,  0.0f,  0.0f,  0.0f,  0.0f,  0.0f,  0.0f},
        {  0.5f,  1.0f,  1.0f,  1.0f,  1.0f,  1.0f,  1.0f,  0.5f},
        { -0.5f,  0.0f,  0.0f,  0.0f,  0.0f,  0.0f,  0.0f, -0.5f},
        { -0.5f,  0.0f,  0.0f,  0.0f,  0.0f,  0.0f,  0.0f, -0.5f},
        { -0.5f,  0.0f,  0.0f,  0.0f,  0.0f,  0.0f,  0.0f, -0.5f},
        { -0.5f,  0.0f,  0.0f,  0.0f,  0.0f,  0.0f,  0.0f, -0.5f},
        { -0.5f,  0.0f,  0.0f,  0.0f,  0.0f,  0.0f,  0.0f, -0.5f},
        {  0.0f,  0.0f,  0.0f,  0.5f,  0.5f,  0.0f,  0.0f,  0.0f}
    };

    private static float[,] _rookEvalBlack = ReverseArray(_rookEvalWhite);

    private static float[,] _evalQueen = {
        { -2.0f, -1.0f, -1.0f, -0.5f, -0.5f, -1.0f, -1.0f, -2.0f},
        { -1.0f,  0.0f,  0.0f,  0.0f,  0.0f,  0.0f,  0.0f, -1.0f},
        { -1.0f,  0.0f,  0.5f,  0.5f,  0.5f,  0.5f,  0.0f, -1.0f},
        { -0.5f,  0.0f,  0.5f,  0.5f,  0.5f,  0.5f,  0.0f, -0.5f},
        {  0.0f,  0.0f,  0.5f,  0.5f,  0.5f,  0.5f,  0.0f, -0.5f},
        { -1.0f,  0.5f,  0.5f,  0.5f,  0.5f,  0.5f,  0.0f, -1.0f},
        { -1.0f,  0.0f,  0.5f,  0.0f,  0.0f,  0.0f,  0.0f, -1.0f},
        { -2.0f, -1.0f, -1.0f, -0.5f, -0.5f, -1.0f, -1.0f, -2.0f}
    };

    private static float[,] _kingEvalWhite = new float[8, 8] {
        { -3.0f, -4.0f, -4.0f, -5.0f, -5.0f, -4.0f, -4.0f, -3.0f},
        { -3.0f, -4.0f, -4.0f, -5.0f, -5.0f, -4.0f, -4.0f, -3.0f},
        { -3.0f, -4.0f, -4.0f, -5.0f, -5.0f, -4.0f, -4.0f, -3.0f},
        { -3.0f, -4.0f, -4.0f, -5.0f, -5.0f, -4.0f, -4.0f, -3.0f},
        { -2.0f, -3.0f, -3.0f, -4.0f, -4.0f, -3.0f, -3.0f, -2.0f},
        { -1.0f, -2.0f, -2.0f, -2.0f, -2.0f, -2.0f, -2.0f, -1.0f},
        {  2.0f,  2.0f,  0.0f,  0.0f,  0.0f,  0.0f,  2.0f,  2.0f},
        {  2.0f,  3.0f,  1.0f,  0.0f,  0.0f,  1.0f,  3.0f,  2.0f}
    };

    private static float[,] _kingEvalBlack = ReverseArray(_kingEvalWhite);

    // Start is called before the first frame update
    void Start()
    {
        _chessManager = GetComponent<OH_BOI_ChessManager>();
    }

    public void MakeBestMove(ChessGame game) {
        StartCoroutine(Minimax(MAX_DEPTH, game));
    }

    private IEnumerator Minimax(int depth, ChessGame game)
    {
        int initialDepth = depth;
        ReadOnlyCollection<Move>[] newGameMovesArr = new ReadOnlyCollection<Move>[depth + 1];
        float[] bestMoveArr = new float[depth + 1];
        float[] alphaArr = new float[depth + 1];
        float[] betaArr = new float[depth + 1];
        Move[] bestMoveFoundArr = new Move[depth + 1];
        int[] newGameMoveIdxArr = new int[depth + 1];

        _passCount = 0;
        newGameMovesArr[depth] = game.GetValidMoves(game.WhoseTurn);
        newGameMoveIdxArr[depth] = 0;
        //bestMoveArr[depth] = -9999;
        //alphaArr[depth] = -10000;
        //betaArr[depth] = 10000;
        bestMoveArr[depth] = float.NegativeInfinity;
        alphaArr[depth]    = float.NegativeInfinity;
        betaArr[depth]     = float.PositiveInfinity;

        while (depth <= initialDepth) {
            _passCount++;
            _passesSinceYield++;

            // This if statement lets Unity continue and picks up the algorithm on the next frame.
            if (_passesSinceYield == 8 || Time.deltaTime > MAX_WAIT) {
                yield return null;
                _passesSinceYield = 0;
            }

            if (DEBUG && _passCount % 128 == 0) Debug.Log("_passCount: " + _passCount);

            // Set the bestMove value to be accessed in the next pass
            if (depth == 0) {
                bestMoveArr[0] = -EvaluateBoard(game);
                depth++;
            }

            // If after zeroth child, update best move, game, alpha/beta
            if (newGameMoveIdxArr[depth] > 0) {
                bool isMaximizingPlayer = (initialDepth - depth) % 2 == 0;
                game.Undo();

                if (isMaximizingPlayer && bestMoveArr[depth - 1] > bestMoveArr[depth] ||
                    !isMaximizingPlayer && bestMoveArr[depth - 1] < bestMoveArr[depth]) {
                    bestMoveArr[depth] = bestMoveArr[depth - 1];
                    bestMoveFoundArr[depth] = newGameMovesArr[depth][newGameMoveIdxArr[depth] - 1];
                }

                if (isMaximizingPlayer)
                    alphaArr[depth] = Mathf.Max(alphaArr[depth], bestMoveArr[depth]);
                else
                    betaArr[depth] = Mathf.Min(betaArr[depth], bestMoveArr[depth]);

                if (betaArr[depth] <= alphaArr[depth]) newGameMoveIdxArr[depth] = newGameMovesArr[depth].Count;
            }
            
            // If not all newGameMoves have been explored, explore one
            if (newGameMoveIdxArr[depth] < newGameMovesArr[depth].Count) {
                game.MakeMove(newGameMovesArr[depth][newGameMoveIdxArr[depth]], true);
                newGameMoveIdxArr[depth]++;
                depth--;
                newGameMovesArr[depth] = game.GetValidMoves(game.WhoseTurn);
                newGameMoveIdxArr[depth] = 0;
                bestMoveArr[depth] = (initialDepth - depth) % 2 == 0 ? float.NegativeInfinity : float.PositiveInfinity;
                alphaArr[depth] = alphaArr[depth + 1];
                betaArr[depth] = betaArr[depth + 1];
            } // Otherwise, wrap up the layer
            else {
                depth++;
            }
        }

        if (DEBUG) Debug.Log("final _passCount: " + _passCount + "\nbestMoveFound: " + bestMoveFoundArr[initialDepth]);

        Move bestMove = bestMoveFoundArr[initialDepth];
        _chessManager.MakeMove(bestMove.OriginalPosition.ToString(), bestMove.NewPosition.ToString(), (int)(bestMove.Player));
    }

    private float EvaluateBoard(ChessGame game) {
        string fen = game.GetFen();
        string list = FenToList(fen);
        fen = fen.Substring(0, fen.IndexOf(' '));
        float totalEvaluation = 0;

        for (int r = 0; r < 8; r++) {
            for (int c = 0; c < 8; c++) {
                totalEvaluation += GetPieceValue(list[8 * r + c], r, c);
            }
        }

        if (game.IsCheckmated(Player.White)) {
            totalEvaluation += -9000;
            Debug.Log("White checkmated with fen " + fen);
        }
        if (game.IsCheckmated(Player.Black)) {
            totalEvaluation += +9000;
            Debug.Log("Black checkmated with fen " + fen);
        }

        return totalEvaluation;
    }

    private static float[,] ReverseArray(float[,] array) {
        float[,] reversed = new float[8,8];

        for (int r = 0; r < 8; r++) {
            for (int c = 0; c < 8; c++) {
                reversed[r, c] = array[7 - r, c];
            }
        }

        return reversed;
    }

    private static string FenToList(string fen) {
        string list = "";

        fen = fen.Substring(0, fen.IndexOf(' '));

        for (int c = 0; c < fen.Length; c++) {
            if (fen[c] != '/') {
                if (char.IsDigit(fen[c])) {
                    var empties = char.GetNumericValue(fen[c]);

                    for (int e = 0; e < empties; e++) {
                        list += " ";
                    }
                } else {
                    list += fen[c];
                }
            }
        }

        return list;
    }

    private float GetPieceValue(char piece, int r, int c) {
        var absoluteValue = GetAbsoluteValue(piece, char.IsUpper(piece), r, c);
        return char.IsUpper(piece) ? absoluteValue : -absoluteValue;
    }

    private static float GetAbsoluteValue(char piece, bool isWhite, int r, int c) {
        switch (char.ToLower(piece)) {
            case 'p':
                return 10 + (isWhite ? _pawnEvalWhite[r, c] : _pawnEvalBlack[r, c]);
            case 'r':
                return 50 + (isWhite ? _rookEvalWhite[r, c] : _rookEvalBlack[r, c]);
            case 'n':
                return 30 + _knightEval[r, c];
            case 'b':
                return 30 + (isWhite ? _bishopEvalWhite[r, c] : _bishopEvalBlack[r, c]);
            case 'q':
                return 90 + _evalQueen[r, c];
            case 'k':
                return 900 + (isWhite ? _kingEvalWhite[r, c] : _kingEvalBlack[r, c]);
            default:
                return 0;
        }
    }
}
{% endhighlight %}

[(to top)](#top)

## Roguelike Procedural Generation Scripts

These were used in a small Unity project to demo some procedural generation of a 3D dungeon-like environment.

### `Cell.cs`

{% highlight cs %}
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

namespace RoguelikePG {
    [System.Serializable]
    public class Cell {
        public Vector3 corner1;
        public Vector3 corner2;

        public Cell(Vector3 corner1, Vector3 corner2) {
            this.corner1 = corner1;
            this.corner2 = corner2;
        }
    }
}
{% endhighlight %}

[(to top)](#top)

### `Door.cs`

{% highlight cs %}
using UnityEngine;

namespace RoguelikePG {
    [System.Serializable]
    public class Door {
        public Vector3 location;
        public Direction dir;

        public Door(Vector3 location, Direction dir) {
            this.location = location;
            this.dir = dir;
        }

        public static void GetAdjacentRoomLocAndRot(Door doorA, Vector3 locA, float rotA, Door doorB, out Vector3 locB, out float rotB) {
            rotB = rotA                                        // Start with the rotational offset of Room A,
                + RPGUtil.AngleTo(Direction.North, doorA.dir)  // add the offset for Door A relative to Room A,
                + 180                                          // flip around since Door B points the opposite direction of Door A
                + RPGUtil.AngleTo(doorB.dir, Direction.North); // add the offset for Room B relative to Door B
            rotB %= 360;                                       // Reestablish rotB within [0, 360)
            rotB = Mathf.RoundToInt(rotB / 90) * 90;           // Scrub any error

            locB = locA; // Start at Room A location
            locB += Quaternion.AngleAxis(rotA, Vector3.up) * doorA.location; // Move to the mutual Door location
            locB -= Quaternion.AngleAxis(rotB, Vector3.up) * doorB.location; // Move to the Room B location
            locB = RPGUtil.RoundVec3(locB); // Scrub any error

            // Thank you NCarter from https://forum.unity.com/threads/rotating-a-vector-by-an-eular-angle.18485/
        }
    }
}
{% endhighlight %}

[(to top)](#top)

### `Room.cs`

{% highlight cs %}
using UnityEngine;
using RoguelikePG;


public class Room : MonoBehaviour {
    public Cell[] Cells;
    // Because of how the openDoors Queue is implemented in RoomManager, it is
    // assumed that a Room has less than 16 doors, although this number can be
    // increased if needed.
    public Door[] Doors;

    //void Start() { } void Update() { }

    /* **************** RoomsOverlap() ****************
     * Determines whether two rooms overlap. A more appropriate name would be
     * "RoomsOverlapOrBlockADoor()", but it's unlikely that the functionality
     * would be needed where Rooms that don't overlap but do block door(s)
     * should return false, so these two operations are combined.
     * 
     * Parameters:
     *   roomA (Room): The first room to evaluate
     *   locA (Vector3): Local position Vector3 of roomA
     *   rotA (float): Local y rotation of roomA
     *   roomB (Room): The second room to evaluate
     *   locB (Vector3): Local position Vector3 of roomB
     *   rotB (float): Local y rotation of roomB
     * 
     * Returns:
     *   bool:
     *     true if there exists shared volume between the Cells of the Rooms
     *     true if there exists some door1 of cell1 that is on cell2 and does
     *       not have a corresponding door2 of cell2 that shares the same
     *       location
     *     false otherwise
     */
    public static bool RoomsOverlap(Room roomA, Vector3 locA, float rotA, Room roomB, Vector3 locB, float rotB) {
        // Potential improvement: keep track of doors on cells with a list of door indices, have out parameter for
        // list of these indices that are now sealed. Should work without it though
        Vector3[] doorsOnCellsA = new Vector3[roomA.Doors.Length];
        Vector3[] doorsOnCellsB = new Vector3[roomB.Doors.Length];

        foreach (Cell cellA in roomA.Cells) {
            Vector3 minCornerA, maxCornerA;
            GetMinAndMaxCorners(cellA, locA, rotA, out minCornerA, out maxCornerA);
            ////Debug.Log("minCornerA: " + minCornerA + ", maxCornerA: " + maxCornerA);

            foreach (Cell cellB in roomB.Cells) {
                Vector3 minCornerB, maxCornerB;
                GetMinAndMaxCorners(cellB, locB, rotB, out minCornerB, out maxCornerB);
                ////Debug.Log("minCornerB: " + minCornerB + ", maxCornerB: " + maxCornerB);

                if (Mathf.Max(minCornerA.x, minCornerB.x) < Mathf.Min(maxCornerA.x, maxCornerB.x) &&
                    Mathf.Max(minCornerA.y, minCornerB.y) < Mathf.Min(maxCornerA.y, maxCornerB.y) &&
                    Mathf.Max(minCornerA.z, minCornerB.z) < Mathf.Min(maxCornerA.z, maxCornerB.z))
                {
                    return true;
                }

                for (int dA = 0; dA < roomA.Doors.Length; dA++) {
                    Vector3 doorOnCellA;

                    if (DoorIsOnCell(roomA.Doors[dA], locA, rotA, cellB, locB, rotB, out doorOnCellA)) {
                        doorsOnCellsA[dA] = doorOnCellA;
                    }
                }
            }

            for (int dB = 0; dB < roomB.Doors.Length; dB++) {
                Vector3 doorOnCellB;

                if (DoorIsOnCell(roomB.Doors[dB], locB, rotB, cellA, locA, rotA, out doorOnCellB)) {
                    doorsOnCellsB[dB] = doorOnCellB;
                }
            }
        }

        for (int dA = 0; dA < roomA.Doors.Length; dA++) {
            for (int dB = 0; dB < roomB.Doors.Length; dB++) {
                if (doorsOnCellsA[dA] != Vector3.zero && doorsOnCellsA[dA] == doorsOnCellsB[dB]) {
                    doorsOnCellsA[dA] = Vector3.zero;
                    doorsOnCellsB[dB] = Vector3.zero;
                }

                if (dA == roomA.Doors.Length - 1 && doorsOnCellsB[dB] != Vector3.zero) return true;
            }

            if (doorsOnCellsA[dA] != Vector3.zero) return true;
        }

        return false;
    }

    /* **************** GetMinAndMaxCorners() ****************
     * Void method that calculates the min and max corners of a Cell, taking
     * into account the Cell's Room's location and rotation.
     * 
     * Parameters:
     *   cell (Cell): Cell to evaluate
     *   loc (Vector3): Local position Vector3 of the Room that the Cell is in
     *   rot (float): Local y rotation of the Room that the Cell is in
     * 
     * Out Parameters:
     *   minCorner (Vector3Int): Vector3Int with the minimum x, y, and z values
     *     out of the corner Vector3s of the Cell, adjusted for location and
     *     rotation
     *   maxCorner (Vector3Int): Vector3Int with the maximum x, y, and z values
     *     out of the corner Vector3s of the Cell, adjusted for location and
     *     rotation
     */
    private static void GetMinAndMaxCorners(Cell cell, Vector3 loc, float rot, out Vector3 minCorner, out Vector3 maxCorner) {
        Vector3 corner1 = RPGUtil.RoundVec3(
            loc + Quaternion.AngleAxis(rot, Vector3.up) * cell.corner1);
        Vector3 corner2 = RPGUtil.RoundVec3(
            loc + Quaternion.AngleAxis(rot, Vector3.up) * cell.corner2);
        minCorner = RPGUtil.RoundVec3(new Vector3(
            Mathf.Min(corner1.x, corner2.x),
            Mathf.Min(corner1.y, corner2.y),
            Mathf.Min(corner1.z, corner2.z)));
        maxCorner = RPGUtil.RoundVec3(new Vector3(
            Mathf.Max(corner1.x, corner2.x),
            Mathf.Max(corner1.y, corner2.y),
            Mathf.Max(corner1.z, corner2.z)));
    }

    /* **************** DoorIsOnCell() ****************
     * Determines whether a Door is "on" a Cell--that is to say that the Door's
     * location Vector3 falls somewhere on a vertical surface of the Cell. Be-
     * cause the Door's adjusted location Vector3 is used if the Door is on the
     * Cell, this calculated value is an out parameter.
     * 
     * Parameters:
     *   door (Door): Door to evaluate
     *   doorRoomLoc (Vector3): Local position Vector3 of the Room that the
     *     Door is in
     *   doorRoomRot (float): Local y rotation of the Room that the Door is in
     *   cell (Cell): Cell to evaluate
     *   cellRoomLoc (Vector3): Local position Vector3 of the Room that the
     *     Cell is in
     *   cellRoomRot (float): Local y rotation of the Room that the Cell is in
     * 
     * Out Parameters:
     *   doorLoc (Vector3): Local position Vector3 of the Door relative to the
     *     it's Room's parent Transform
     * 
     * Returns:
     *   bool: true iff Door is on one of the vertical planes of the Cell
     */
    private static bool DoorIsOnCell(Door door, Vector3 doorRoomLoc, float doorRoomRot, Cell cell, Vector3 cellRoomLoc, float cellRoomRot, out Vector3 doorLoc) {
        doorLoc = doorRoomLoc + Quaternion.AngleAxis(doorRoomRot, Vector3.up) * door.location;
        doorLoc = RPGUtil.RoundVec3(doorLoc, 1);
        Vector3 minCorner, maxCorner;
        GetMinAndMaxCorners(cell, cellRoomLoc, cellRoomRot, out minCorner, out maxCorner);

        /*
         * Hopefully the spacing of this return statements help clarify what it does, but in case it doesn't:
         * First and foremost, the door must fall in the cell's y range, since all doors are perpendicular to the XZ plane
         * There are two cases for the next check:
         *      Case 1: The door is parallel to the YZ plane
         *          The door must then be on the x boundary of the cell (either the min or max),
         *          and the door must fall in the cell's z range
         *      Case 2: The door is parallel to the XY plane
         *          The door must then be on the z boundary of the cell (either the min or max)
         *          and the door must fall in the cell's x range
         */

        return
            minCorner.y < doorLoc.y
            &&
            doorLoc.y < maxCorner.y
            &&
            (
                (
                    (
                        doorLoc.x == minCorner.x
                        ||
                        doorLoc.x == maxCorner.x
                    )
                    &&
                    minCorner.z < doorLoc.z
                    &&
                    doorLoc.z < maxCorner.z
                )
                ||
                (
                    (
                        doorLoc.z == minCorner.z
                        ||
                        doorLoc.z == maxCorner.z
                    )
                    &&
                    minCorner.x < doorLoc.x
                    &&
                    doorLoc.x < maxCorner.x
                )
            );
    }
}
{% endhighlight %}

[(to top)](#top)

### `RoomManager.cs`

This was partner-coded with a classmate, Lauren Gray.

{% highlight cs %}
using System.Collections.Generic;
using UnityEngine;
using RoguelikePG;

public class RoomManager : MonoBehaviour {
    public GameObject[] prefabs;

    /*
     * These Lists keep track of which rooms are "active"--those which have
     * already been added to the structure. These rooms aren't necessarily in
     * the scene, but all the data necessary to instantiate them is saved,
     * hence the three lists. These were originally stored as one List of
     * Tuples, but Unity's lack of Tuple support made that difficult. The
     * PrefabIndices List keeps track of the index of prefabs used for a room.
     * RoomPoses keeps track of the Vector3 local positions of the rooms.
     * RoomRots keeps track of the y rotation of the rooms.
     */
    private List<int> _activeRoomPrefabIndices;
    private List<Vector3> _activeRoomPoses;
    private List<float> _activeRoomRots;

    // Start is called before the first frame update
    void Start() {
        _activeRoomPrefabIndices = new List<int>();
        _activeRoomPoses = new List<Vector3>();
        _activeRoomRots = new List<float>();

        _activeRoomPrefabIndices.Add(0);
        _activeRoomPoses.Add(Vector3.zero);
        _activeRoomRots.Add(0);

        /*
         * openDoors is a Queue of ints, where each int represents the index to
         * an "open" door (one that isn't coupled with the door of another
         * room) in an indexable room of the _activeRoom Lists. The two indices
         * are stored in the following manner, given the 32 bit signed Queue
         * members: 0x rr rr rr rd. The first 7 nibbles (excluding the leading
         * sign bit) are for the index of the room in the _activeRoom Lists.
         * The last nibble is for the Door's index within the indexed Room.
         */
        Queue<int> openDoors = new Queue<int>();

        for (int d = 0; d < prefabs[0].GetComponent<Room>().Doors.Length; d++) {
            openDoors.Enqueue(d);
        }

        int randRoom;
        int randDoor;
        int startRoom;
        int startDoor;
        bool matchFound;

        while (_activeRoomPrefabIndices.Count < 20 && openDoors.Count > 0) {
            int currDoorData = openDoors.Dequeue();
            int activeRoomsIdx = currDoorData >> 4, currDoorIdx = currDoorData & 0xF;
            int currRoomPrefabIdx = _activeRoomPrefabIndices[activeRoomsIdx];
            //Room currRoom = prefabs[currRoomPrefabIdx].GetComponent<Room>();
            Vector3 currRoomLoc = _activeRoomPoses[activeRoomsIdx];
            float currRoomRot = _activeRoomRots[activeRoomsIdx];
            Door currDoor = prefabs[currRoomPrefabIdx].GetComponent<Room>().Doors[currDoorIdx];


            // pick a random room prefab and remember this is the index you started on
            randRoom = Random.Range(0, prefabs.Length);
            startRoom = randRoom;

            do {
                Room nextRoom = prefabs[randRoom].GetComponent<Room>();
                Vector3 nextRoomLoc;
                float nextRoomRot;

                // pick a random door and remember this is the index you started on
                randDoor = Random.Range(0, nextRoom.Doors.Length);
                startDoor = randDoor;

                do {
                    matchFound = true;
                    Door nextDoor = nextRoom.Doors[randDoor];
                    Door.GetAdjacentRoomLocAndRot(currDoor, currRoomLoc, currRoomRot,
                        nextDoor, out nextRoomLoc, out nextRoomRot);

                    for (int ar = 0; ar < _activeRoomPrefabIndices.Count; ar++) {
                        int activeRoomPrefabIdx = _activeRoomPrefabIndices[ar];
                        Vector3 activeRoomLoc = _activeRoomPoses[ar];
                        float activeRoomRot = _activeRoomRots[ar];
                        Room activeRoom = prefabs[activeRoomPrefabIdx].GetComponent<Room>();

                        if (Room.RoomsOverlap(activeRoom, activeRoomLoc, activeRoomRot, nextRoom, nextRoomLoc, nextRoomRot)) {
                            matchFound = false;
                            break;
                        }
                    }

                    if (matchFound) {
                        _activeRoomPrefabIndices.Add(randRoom);
                        _activeRoomPoses.Add(nextRoomLoc);
                        _activeRoomRots.Add(nextRoomRot);

                        // add open doors to open doors
                        for (int d = 0; d < nextRoom.Doors.Length; d++) {
                            if (d != randDoor) {
                                openDoors.Enqueue(((_activeRoomPrefabIndices.Count - 1) << 4) | d);
                            }
                        }

                        break;
                    }

                    randDoor++;
                    randDoor %= nextRoom.Doors.Length;
                } while (randDoor != startDoor && !matchFound);

                randRoom++;
                randRoom %= prefabs.Length;
            } while (randRoom != startRoom && !matchFound);
        }

        for(int ar = 0; ar < _activeRoomPrefabIndices.Count; ar++) {
            int activeRoomPrefabIdx = _activeRoomPrefabIndices[ar];
            Vector3 activeRoomLoc = _activeRoomPoses[ar];
            float activeRoomRot = _activeRoomRots[ar];
            GameObject activeRoomGO = Object.Instantiate(prefabs[activeRoomPrefabIdx]);
            activeRoomGO.transform.localPosition = activeRoomLoc;
            activeRoomGO.transform.Rotate(Vector3.up, activeRoomRot);
        }
    }

    // Update is called once per frame void Update() { }
}
{% endhighlight %}

[(to top)](#top)

### `RPGUtil.cs`

{% highlight cs %}
using System;
using UnityEngine;

namespace RoguelikePG
{
    [System.Serializable]
    public enum Direction : byte
    {
        East,
        North,
        West,
        South
    }

    public class RPGUtil
    {
        public static float AngleTo(Direction from, Direction to) {
            return 90 * ((from - to) % 4);
        }

        public static string Vector3ArrayString(Vector3[] vector3s) {
            string retStr = "[";

            for (int i = 0; i < vector3s.Length; i++) {
                if (i != 0) retStr += ", ";
                retStr += vector3s[i];
            }

            return retStr + "]";
        }

        public static string Vector3LongString(Vector3 vector3)
        {
            return String.Format("<{0:F12}, {1:F12}, {2:F12}>", vector3.x, vector3.y, vector3.z);
        }

        public static Vector3 RoundVec3(Vector3 vector3, int bits) {
            return RoundVec3(vector3 * (1 << bits)) / (1 << bits);
        }

        public static Vector3 RoundVec3(Vector3 vector3) {
            vector3.x = Mathf.RoundToInt(vector3.x);
            vector3.y = Mathf.RoundToInt(vector3.y);
            vector3.z = Mathf.RoundToInt(vector3.z);
            return vector3;
        }
    }
}
{% endhighlight %}

[(to top)](#top)

[Go to the Home Page]({{ '/' | absolute_url }})
