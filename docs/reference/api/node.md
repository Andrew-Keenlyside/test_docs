# Node Class

The `Node` class represents a single fiber candidate point in the BRIDGE pipeline.

## Class Definition

```python
from bridge.objs.Node import Node
```

## Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `id` | int | Unique node identifier within chunk |
| `_centre` | np.ndarray (float32, shape=3) | Spatial coordinates [z, y, x] |
| `principal_eigenvector` | np.ndarray (float32, shape=3) | Fiber orientation vector |
| `img` | int (0-255) | Image intensity value |
| `fa` | float32 | Fractional anisotropy |
| `local_z` | float32 | Local z-score |
| `is_endpoint` | bool | Whether node is a path endpoint |
| `searched` | bool | Whether node has been searched in BFS |

## Binary Format

Nodes are serialized using a fixed-size binary format for efficient storage:

```python
NODE_STRUCT_FMT = '<i3f3fBff??'
# i     = id (int32)
# 3f    = centre (3 x float32)
# 3f    = principal_eigenvector (3 x float32)
# B     = img (uint8)
# f     = fa (float32)
# f     = local_z (float32)
# ?     = is_endpoint (bool)
# ?     = searched (bool)

NODE_RECORD_SIZE = 37  # bytes
```

## Methods

### `__init__(self, id=None)`

Create a new Node instance.

```python
node = Node(id=42)
node._centre = np.array([100.5, 200.3, 150.7], dtype=np.float32)
node.principal_eigenvector = np.array([0.1, 0.2, 0.97], dtype=np.float32)
node.fa = 0.45
```

### `to_binary(self) -> bytes`

Serialize node to binary format.

```python
binary_data = node.to_binary()
assert len(binary_data) == 37
```

### `from_binary(cls, buffer: bytes) -> Node`

Deserialize node from binary format.

```python
node = Node.from_binary(binary_data)
```

### `pack_into(self, buf: bytearray, offset: int) -> None`

Pack node into an existing buffer at specified offset. More efficient for batch operations.

```python
buf = bytearray(37 * 100)  # Space for 100 nodes
for i, node in enumerate(nodes):
    node.pack_into(buf, i * 37)
```

### `to_dict(self) -> dict`

Convert node to dictionary for JSON serialization.

```python
d = node.to_dict()
# {
#     "id": 42,
#     "centre": [100.5, 200.3, 150.7],
#     "principal_eigenvector": [0.1, 0.2, 0.97],
#     "img": 128,
#     "fa": 0.45,
#     "local_z": 1.2,
#     "is_endpoint": True,
#     "searched": False
# }
```

### `from_dict(cls, data: dict) -> Node`

Create node from dictionary.

```python
node = Node.from_dict(d)
```

## Usage Examples

### Loading Nodes from File

```python
from bridge.utils.utils_nodes import load_all_nodes_with_metadata

nodes, metadata = load_all_nodes_with_metadata(
    "network/nodes/sub-01_res-0.02mm_cid-0-0-0_nodes.bin"
)

for node in nodes[:5]:
    print(f"Node {node.id}: centre={node._centre}, FA={node.fa:.3f}")
```

### Creating Nodes Programmatically

```python
import numpy as np
from bridge.objs.Node import Node

# Create a node
node = Node(id=0)
node._centre = np.array([50.0, 100.0, 75.0], dtype=np.float32)
node.principal_eigenvector = np.array([0.0, 0.0, 1.0], dtype=np.float32)
node.fa = 0.6
node.local_z = 1.5
node.img = 200
node.is_endpoint = True
node.searched = False
```

## File Format

Node files follow this structure:

```
[4 bytes]  Magic number: b'NOD3'
[4 bytes]  Header size (uint32, little-endian)
[N bytes]  Header JSON
[M * 37]   Node records (M nodes, 37 bytes each)
```

Header JSON example:
```json
{
  "record_size": 37,
  "node_count": 15234,
  "metadata": {
    "chunk_id": "0-0-0",
    "origin": [0, 0, 0],
    "chunk_dims": [256, 256, 256]
  }
}
```

## See Also

- [Graph Class](graph.md)
- [Core 1: Node Extraction](core1.md)
- [File Formats](../file-formats.md)
