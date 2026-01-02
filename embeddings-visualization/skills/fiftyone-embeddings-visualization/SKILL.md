---
name: fiftyone-embeddings-visualization
description: Visualize datasets in 2D using embeddings with UMAP or t-SNE dimensionality reduction. Use when users want to explore dataset structure, find clusters in images, identify outliers, color samples by class or metadata, or understand data distribution. Requires FiftyOne MCP server with @voxel51/brain plugin installed.
---

# Embeddings Visualization in FiftyOne

## Overview

Visualize your dataset in 2D using deep learning embeddings and dimensionality reduction (UMAP/t-SNE). Explore clusters, find outliers, and color samples by any field.

**Use this skill when:**
- Visualizing dataset structure in 2D
- Finding natural clusters in images
- Identifying outliers or anomalies
- Exploring data distribution by class or metadata
- Understanding embedding space relationships

## Prerequisites

- FiftyOne MCP server installed and running
- `@voxel51/brain` plugin installed and enabled
- Dataset with image samples loaded in FiftyOne

## Key Directives

**ALWAYS follow these rules:**

### 1. Set context first
```python
set_context(dataset_name="my-dataset")
```

### 2. Launch FiftyOne App
Brain operators are delegated and require the app:
```python
launch_app()
```
Wait 5-10 seconds for initialization.

### 3. Discover operators dynamically
```python
# List all brain operators
list_operators(builtin_only=False)

# Get schema for specific operator
get_operator_schema(operator_uri="@voxel51/brain/compute_visualization")
```

### 4. Compute embeddings before visualization
Embeddings are required for dimensionality reduction:
```python
execute_operator(
    operator_uri="@voxel51/brain/compute_similarity",
    params={"brain_key": "img_sim", "model": "clip-vit-base32-torch"}
)
```

### 5. Close app when done
```python
close_app()
```

## Complete Workflow

### Step 1: Setup
```python
# Set context
set_context(dataset_name="my-dataset")

# Launch app (required for brain operators)
launch_app()
```

### Step 2: Verify Brain Plugin
```python
# Check if brain plugin is available
list_plugins(enabled=True)

# If not installed:
download_plugin(
    url_or_repo="voxel51/fiftyone-plugins",
    plugin_names=["@voxel51/brain"]
)
enable_plugin(plugin_name="@voxel51/brain")
```

### Step 3: Discover Brain Operators
```python
# List all available operators
list_operators(builtin_only=False)

# Get schema for compute_visualization
get_operator_schema(operator_uri="@voxel51/brain/compute_visualization")
```

### Step 4: Compute Embeddings
```python
# Execute operator to compute embeddings
execute_operator(
    operator_uri="@voxel51/brain/compute_similarity",
    params={
        "brain_key": "img_viz",
        "model": "clip-vit-base32-torch"
    }
)
```

**Recommended embedding models:**
- `clip-vit-base32-torch` - Best for general visual + semantic similarity
- `dinov2-vits14-torch` - Best for visual similarity only
- `mobilenet-v2-imagenet-torch` - Fast, lightweight option

### Step 5: Compute 2D Visualization
```python
execute_operator(
    operator_uri="@voxel51/brain/compute_visualization",
    params={
        "brain_key": "img_viz",
        "method": "umap",
        "num_dims": 2
    }
)
```

**Dimensionality reduction methods:**
- `umap` - (Recommended) Preserves local and global structure, faster
- `tsne` - Better local structure, slower on large datasets
- `pca` - Linear reduction, fastest but less informative

### Step 6: View Embeddings Panel in App

After computing visualization, open the FiftyOne App at http://localhost:5151/ and:

1. Click the **Embeddings** panel icon (scatter plot) in the top toolbar
2. Select the brain key (e.g., `img_viz`) from the dropdown
3. Points represent samples in 2D embedding space
4. Click points to select samples, lasso to select groups

### Step 7: Color by Field

To color points by a specific field (class, tag, or metadata):

**Option A: Color by classification field**
```python
# In the App Embeddings panel:
# Use the "Color by" dropdown to select a field like "ground_truth" or "predictions"
```

**Option B: Use set_view to filter and explore**
```python
# Filter to specific class
set_view(filters={"ground_truth.label": "dog"})

# Filter by tag
set_view(tags=["validated"])

# Clear filter to show all
clear_view()
```

### Step 8: Find Outliers

Outliers appear as isolated points far from clusters:

```python
# Compute uniqueness scores (higher = more unique/outlier)
execute_operator(
    operator_uri="@voxel51/brain/compute_uniqueness",
    params={
        "brain_key": "img_viz"
    }
)

# View most unique samples (potential outliers)
set_view(sort_by="uniqueness", reverse=True, limit=50)
```

### Step 9: Find Clusters

Use the App's Embeddings panel to visually identify clusters, then:

**Option A: Lasso selection in App**
1. Use lasso tool to select a cluster
2. Selected samples are highlighted
3. Tag or export selected samples

**Option B: Use similarity to find cluster members**
```python
# Sort by similarity to a representative sample
execute_operator(
    operator_uri="@voxel51/brain/sort_by_similarity",
    params={
        "brain_key": "img_viz",
        "query_id": "sample_id_from_cluster",
        "k": 100
    }
)
```

### Step 10: Clean Up
```python
close_app()
```

## Available Tools

### Session View Tools

| Tool | Description |
|------|-------------|
| `set_view(filters={...})` | Filter samples by field values |
| `set_view(tags=[...])` | Filter samples by tags |
| `set_view(sort_by="...", reverse=True)` | Sort samples by field |
| `set_view(limit=N)` | Limit to N samples |
| `clear_view()` | Clear filters, show all samples |

### Brain Operators for Visualization

Use `list_operators()` to discover and `get_operator_schema()` to see parameters:

| Operator | Description |
|----------|-------------|
| `@voxel51/brain/compute_similarity` | Compute embeddings and similarity index |
| `@voxel51/brain/compute_visualization` | Reduce embeddings to 2D/3D for visualization |
| `@voxel51/brain/compute_uniqueness` | Score samples by uniqueness (outlier detection) |
| `@voxel51/brain/sort_by_similarity` | Sort by similarity to a query sample |

## Common Use Cases

### Use Case 1: Basic Dataset Exploration
Visualize dataset structure and explore clusters:
```python
set_context(dataset_name="my-dataset")
launch_app()

# Compute embeddings
execute_operator(
    operator_uri="@voxel51/brain/compute_similarity",
    params={"brain_key": "exploration", "model": "clip-vit-base32-torch"}
)

# Generate 2D visualization
execute_operator(
    operator_uri="@voxel51/brain/compute_visualization",
    params={
        "brain_key": "exploration",
        "method": "umap",
        "num_dims": 2
    }
)

# Direct user to App Embeddings panel
# Color by ground_truth or predictions field

close_app()
```

### Use Case 2: Find Outliers in Dataset
Identify anomalous or mislabeled samples:
```python
set_context(dataset_name="my-dataset")
launch_app()

# Compute embeddings
execute_operator(
    operator_uri="@voxel51/brain/compute_similarity",
    params={"brain_key": "outliers", "model": "clip-vit-base32-torch"}
)

# Compute uniqueness scores
execute_operator(
    operator_uri="@voxel51/brain/compute_uniqueness",
    params={"brain_key": "outliers"}
)

# Generate visualization
execute_operator(
    operator_uri="@voxel51/brain/compute_visualization",
    params={
        "brain_key": "outliers",
        "method": "umap",
        "num_dims": 2
    }
)

# View top outliers
set_view(sort_by="uniqueness", reverse=True, limit=100)

close_app()
```

### Use Case 3: Compare Classes in Embedding Space
See how different classes cluster:
```python
set_context(dataset_name="my-dataset")
launch_app()

# Compute embeddings
execute_operator(
    operator_uri="@voxel51/brain/compute_similarity",
    params={"brain_key": "class_viz", "model": "clip-vit-base32-torch"}
)

# Generate visualization
execute_operator(
    operator_uri="@voxel51/brain/compute_visualization",
    params={
        "brain_key": "class_viz",
        "method": "umap",
        "num_dims": 2
    }
)

# In App: Color by classification field (ground_truth or predictions)
# Look for:
# - Well-separated clusters = good class distinction
# - Overlapping clusters = similar classes or confusion
# - Scattered points = high variance within class

close_app()
```

### Use Case 4: Analyze Model Predictions
Compare ground truth vs predictions in embedding space:
```python
set_context(dataset_name="my-dataset")
launch_app()

# Compute embeddings
execute_operator(
    operator_uri="@voxel51/brain/compute_similarity",
    params={"brain_key": "pred_analysis", "model": "clip-vit-base32-torch"}
)

# Generate visualization
execute_operator(
    operator_uri="@voxel51/brain/compute_visualization",
    params={
        "brain_key": "pred_analysis",
        "method": "umap",
        "num_dims": 2
    }
)

# In App Embeddings panel:
# 1. Color by ground_truth - see true class distribution
# 2. Color by predictions - see model's view
# 3. Look for mismatches to find errors

close_app()
```

### Use Case 5: t-SNE for Publication-Quality Plots
Use t-SNE for better local structure (slower):
```python
set_context(dataset_name="my-dataset")
launch_app()

execute_operator(
    operator_uri="@voxel51/brain/compute_similarity",
    params={"brain_key": "tsne_viz", "model": "dinov2-vits14-torch"}
)

execute_operator(
    operator_uri="@voxel51/brain/compute_visualization",
    params={
        "brain_key": "tsne_viz",
        "method": "tsne",
        "num_dims": 2
    }
)

close_app()
```

## Troubleshooting

**Error: "No executor available"**
- Cause: Delegated operators require the App executor
- Solution: Ensure `launch_app()` was called and wait 5-10 seconds

**Error: "Brain key not found"**
- Cause: Embeddings not computed
- Solution: Run `compute_similarity` first with a `brain_key`

**Error: "Operator not found"**
- Cause: Brain plugin not installed
- Solution: Install with `download_plugin()` and `enable_plugin()`

**Error: "Missing dependency" (e.g., umap-learn, sklearn)**
- The MCP server detects missing dependencies automatically
- Response includes `missing_package` and `install_command`
- Example response:
  ```json
  {
    "error_type": "missing_dependency",
    "missing_package": "umap-learn",
    "install_command": "pip install umap-learn"
  }
  ```
- Offer to run the install command for the user
- After installation, restart MCP server and retry

**Visualization is slow**
- Use UMAP instead of t-SNE for large datasets
- Use faster embedding model: `mobilenet-v2-imagenet-torch`
- Process subset first: `set_view(limit=1000)`

**Embeddings panel not showing**
- Ensure visualization was computed (not just embeddings)
- Check brain_key matches in both compute_similarity and compute_visualization
- Refresh the App page

**Points not colored correctly**
- Verify the field exists on samples
- Check field type is compatible (Classification, Detections, or string)

## Best Practices

1. **Discover dynamically** - Use `list_operators()` and `get_operator_schema()` to get current operator names and parameters
2. **Choose the right model** - CLIP for semantic similarity, DINOv2 for visual similarity
3. **Start with UMAP** - Faster and often better than t-SNE for exploration
4. **Use uniqueness for outliers** - More reliable than visual inspection alone
5. **Store embeddings** - Reuse for multiple visualizations via `brain_key`
6. **Subset large datasets** - Compute on subset first, then full dataset

## Performance Notes

**Embedding computation time:**
- 1,000 images: ~1-2 minutes
- 10,000 images: ~10-15 minutes
- 100,000 images: ~1-2 hours

**Visualization computation time:**
- UMAP: ~30 seconds for 10,000 samples
- t-SNE: ~5-10 minutes for 10,000 samples
- PCA: ~5 seconds for 10,000 samples

**Memory requirements:**
- ~2KB per image for embeddings
- ~16 bytes per image for 2D coordinates

## Resources

- [FiftyOne Brain Documentation](https://docs.voxel51.com/user_guide/brain.html)
- [Visualizing Embeddings Guide](https://docs.voxel51.com/user_guide/embeddings.html)
- [Brain Plugin Source](https://github.com/voxel51/fiftyone-plugins/tree/main/plugins/brain)

## License

Copyright 2017-2025, Voxel51, Inc.
Apache 2.0 License
