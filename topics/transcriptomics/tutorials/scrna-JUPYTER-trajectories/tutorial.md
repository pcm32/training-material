---
layout: tutorial_hands_on

title: Trajectory Analysis using Python (Jupyter Notebook) in Galaxy
subtopic: single-cell
priority: 11
zenodo_link: 'https://zenodo.org/record/4655947'
questions:
- How can I infer lineage relationships between single cells based on their RNA, without a time series?
objectives:
- Execute multiple plotting methods designed to maintain lineage relationships between cells
- Interpret these plots
time_estimation: 2H
key_points:
- Trajectory analysis is less robust than pure plotting methods, as such 'inferred relationships' are a bigger mathematical leap
- As always with single-cell analysis, you must know enough biology to deduce if your analysis is reasonable, before exploring or deducing novel insight
requirements:
-
    type: "internal"
    topic_name: transcriptomics
    tutorials:
        - droplet-quantification-preprocessing
        - scrna-seq-basic-pipeline
-
    type: "internal"
    topic_name: galaxy-interface
    tutorials:
        - galaxy-intro-jupyter
tags:
- single-cell
- 10x
contributors:
- nomadscientist
- mtekman

---


# Introduction
{:.no_toc}

You've done all the hard work of preparing a single cell matrix, processing it, plotting it, interpreting it, finding lots of lovely genes, all within the glorious Galaxy interface. Now you want to infer trajectories, or relationships between cells... and you've been threatened with learning Python to do so! Well, fear not. If you can have a run-through of a basic python coding introduction such as [this one](https://www.w3schools.com/python/), then that will help you make more sense of this tutorial, however you'll be able to make and interpret glorious plots even without understanding the Python coding language. This is the beauty of Galaxy - all the 'set-up' is identical across computers, because it's browser based. So fear not!

Traditionally, we thought that differentiating or changing cells jumped between discrete states, so 'Cell A' became 'Cell B' as part of its maturation. However, most data shows otherwise, that generally there is a spectrum (a 'trajectory', if you will...) of small, subtle changes along a pathway of that differentiation. Trying to analyse cells every 10 seconds can be pretty tricky, so 'pseudotime' analysis takes a single sample and assumes that those cells are all on slightly different points along a path of differentiation. Some cells might be slightly more mature and others slightly less, all captured at the same 'time'. We 'assume' or 'infer' relationships between cells.

We will use the same sample from the previous two tutorials, which contains largely T-cells in the thymus. We know T-cells differentiate in the thymus, so we would assume that we would capture cells at slightly different time points within the same sample. Furthermore, our cluster analysis alone showed different states of T-cell. Now it's time to look further!

> ### Agenda
>
> In this tutorial, we will cover:
>
> 1. TOC
> {:toc}
>
{: .agenda}

## Get data

We've provided you with experimental data to analyse from a mouse dataset of fetal growth restriction {% cite Bacon2018 %}. This is the full dataset generated from [this tutorial](https://training.galaxyproject.org/training-material/topics/transcriptomics/tutorials/scrna-seq-basic-pipeline/tutorial.html#neighborhood-graph) (see the study in Single Cell Expression Atlas [here](https://www.ebi.ac.uk/gxa/sc/experiments/E-MTAB-6945/results/tsne) and the project submission [here](https://www.ebi.ac.uk/arrayexpress/experiments/E-MTAB-6945/)). You can find the final dataset in this [input history](https://humancellatlas.usegalaxy.eu/u/wendi.bacon.training/h/trajectories---input) or download from Zenodo below.

> ### {% icon hands_on %} Hands-on: Data upload
>
> 1. Create a new history for this tutorial
> 2. Import the AnnData object from [Zenodo]({{ page.zenodo_link }})
>
>    ```
>    {{ page.zenodo_link }}/files/Trajectories_Instructions.ipynb
>    {{ page.zenodo_link }}/files/Final_cell_annotated_object.h5ad
>    ```
>
>    {% snippet faqs/galaxy/datasets_import_via_link.md %}
>
> 3. **Rename** {% icon galaxy-pencil %} the .h5ad object as `Final cell annotated object`
>
>    {% snippet faqs/galaxy/datasets_rename.md name="Final cell annotated object" %}
>
> 4. Check that the datatype is `h5ad`
>
>    {% snippet faqs/galaxy/datasets_change_datatype.md datatype="h5ad" %}
>
> 5. **Rename** {% icon galaxy-pencil %} the .ipynb object as `Trajectories_Instructions.ipynb`
>
> 6. Check that the datatype is `.ipynb`
>
{: .hands_on}

## Filtering for T-cells

One problem with our current dataset is that it's not just T-cells: we found in the previous tutorial that it also contains macrophages and red blood cells. This is a problem, because trajectory analysis will generally try to find relationships between all the cells in the sample. We need to remove those cell types to analyse the trajectory.

> ### {% icon hands_on %} Hands-on: Removing macrophages and RBCs
>
> 1. {% tool [Manipulate AnnData](toolshed.g2.bx.psu.edu/repos/iuc/anndata_manipulate/anndata_manipulate/0.7.5+galaxy0) %} with the following parameters:
>    - {% icon param-file %} *"Annotated data matrix"*: `Final cell annotated object`
>    - *"Function to manipulate the object"*: `Filter observations or variables`
>    - *"What to filter?"*: `Observations (obs)`
>    - *"Type of filtering?"*: `By key (column) values`
>    - *"Key to filter"*: `cell_type`
>    - *"Type of value to filter"*: `Text`
>    - *"Filter"*: `not equal to`
>    - *"Value"*: `RBC`
>
> 2. {% tool [Manipulate AnnData](toolshed.g2.bx.psu.edu/repos/iuc/anndata_manipulate/anndata_manipulate/0.7.5+galaxy0) %} with the following parameters:
>    - {% icon param-file %} *"Annotated data matrix"*: (output of **Manipulate AnnData** {% icon tool %})
>    - *"Function to manipulate the object"*: `Filter observations or variables`
>    - *"What to filter?"*: `Observations (obs)`
>    - *"Type of filtering?"*: `By key (column) values`
>    - *"Key to filter"*: `cell_type`
>    - *"Type of value to filter"*: `Text`
>    - *"Filter"*: `not equal to`
>    - *"Value"*: `Macrophages`
>
> 3. **Rename** {% icon galaxy-pencil %} output h5ad `T-cell_object.h5ad`
>
{: .hands_on}

You should now have `7795` cells, as opposed to the `7874` you started with. You've only removed a few cells (the contaminants!), but it makes a big difference in the next steps.

Take note of what # this dataset is in your history, as you will need that shortly!

## Launching Jupyter

> ### {% icon hands_on %} Hands-on: Downloading the directions
>
> 1. Click the **Trajectories_Instructions.ipynb** dataset and the download {% icon galaxy-save %} button to get the file onto your computer
>
{: .hands_on}

JupyterLab is a bit like RStudio but for other coding languages. What, you've never heard of [RStudio](https://www.rstudio.com/products/rstudio/features/)? Then don't worry, just follow the instructions!

{% icon warning %} Please note: this is only currently available on the [usegalaxy.eu](https://usegalaxy.eu) and [usegalaxy.org](https://usegalaxy.org) sites.

> ### {% icon hands_on %} Hands-on: Launching JupyterLab
>
> 1. {% tool [Interactive Jupyter Notebook](interactive_tool_jupyter_notebook) %} with the following parameters:
>    - *"Do you already have a notebook?"*: `Start with a fresh notebook`
>    - {% icon param-file %} *"Include data into the environment"*:  `Nothing selected`
>
>    This may take a moment, but once the `Executed notebook` in your dataset is orange, you are up and running!
>
> 4. Either click on the blue `User menu`, or go to the top of the screen and choose `User` and then `Active InteractiveTools`
>
> 5. Click on the newest `Jupyter Interactive Tool`.
>
{: .hands_on}

Welcome!

> ### {% icon warning %} Danger: You can lose data!
> Do NOT delete or close this notebook dataset in your history. YOU WILL LOSE IT!
{: .warning}

> ### {% icon hands_on %} Hands-on: Creating a notebook
>
> 1. Click the **Python 3** icon under **Notebook**
>
>   ![Python 3 icon](../../images/wab-python3logo.png "Python 3 Button")
>
> 2. Save your file (**File**: **Save**, or click the {% icon galaxy-save %} Save icon at the top left)
>
> 3. If you right click on the file in the folder window at the left, you can rename your file `whateveryoulike.ipynb`
>
{: .hands_on}

Cool! Now you know how to create a file! Helpfully, however, we have created one for you, and you've downloaded it onto your computer already!

> ### {% icon hands_on %} Hands-on: Uploading the tutorial notebook
>
> 1. In the folder window, {% icon galaxy-upload %} Upload the `Trajectories_Instructions.ipynb` from your computer. It should appear in the file window.
>
> 2. Open it by double clicking it in the file window.
>
{: .hands_on}

> ### {% icon warning %} You should {% icon galaxy-save %} **Save** frequently!
> This is both for good practice and to protect you in case you accidentally close the browser. Your environment will still run, so it will contain the last saved notebook you have. You might eventually stop your environment after this tutorial, but ONLY once you have saved and exported your notebook (more on that at the end!) Note that you can have multiple notebooks going at the same time within this JupyterLab, so if you do, you will need to save and export each individual notebook. You can also download them at any time.
{: .warning}

# Run the tutorial!

At this point, to prevent you having to switch back and forth between browsers, the directions for the rest of tutorial are all in the notebook you input! Go to `data` and double click `Trajectories_Instructions.ipynb` You may have to change certain numbers in the code blocks, so do read carefully. You will be able to run each step be clicking on the code block and pressing the {% icon workflow-run %} *Run the selected cells and advance* step. You will want to keep a tab open with your Galaxy history showing (so just launch another browser of your usegalaxy.eu instance), so that you can see when your files appear there. The tutorial is adapted from the [Scanpy Trajectory inference tutorial](https://scanpy-tutorials.readthedocs.io/en/latest/paga-paul15.html).

# Tutorial Plot Answers

Just in case, we've put the plots you should generate in the tutorial here. If things have gone wrong, you can also download this [answer key tutorial]({{ page.zenodo_link }}/files/Trajectories_AnswerKey.ipynb).

![Plot1-Force-Directed Graph](../../images/wab-Plot1.png "Plot1-Force-Directed Graph")

![Diffusion Map](../../images/wab-Plot2.png "Diffusion Map")

![PAGA](../../images/wab-Plot3.png "PAGA")

![Force-Directed + PAGA - Cell type](../../images/wab-Plot4.png "Force-Directed + PAGA - Cell type")

![Force-Directed + PAGA - Genotype](../../images/wab-Plot5.png "Force-Directed + PAGA - Genotype")

![Force-Directed + PAGA - Markers](../../images/wab-Plot6.png "Force-Directed + PAGA - Markers")

![Force-Directed + Pseudotime](../../images/wab-Plot7.png "Force-Directed + Pseudotime")

# After Jupyter

{% icon congratulations %} Congratulations! You've made it through Jupyter!

> ### {% icon hands_on %} Hands-on: Closing JupyterLab
>
> 1. Click **User**: **Active Interactive Tools**
>
> 2. Tick {% icon galaxy-selector %} the box of your Jupyter Interactive Tool, and click **Stop**
>
{: .hands_on}

If you want to run this notebook again, or share it with others, it now exists in your history. You can use this 'finished' version just the same way as you downloaded the directions file and uploaded into the Jupyter environment.

# Conclusion
{:.no_toc}

{% icon congratulations %} Congratulations! You've made it to the end! You might be interested in the [Answer Key History](https://humancellatlas.usegalaxy.eu/u/wendi.bacon.training/h/trajectories---jupyter---answer-key)

In this tutorial, you moved from called clusters to inferred relationships and trajectories using pseudotime analysis. You found an alternative to PCA (diffusion map), an alternative to tSNE (force-directed graph), a means of identifying cluster relationships (PAGA), and a metric for pseudotime (diffusion pseudotime) to identify early and late cells. If you were working in a group, you found that such analysis is slightly more sensitive to your decisions than the simpler filtering/plotting/clustering is. We are inferring and assuming relationships and time, so that makes sense!

To discuss with like-minded scientists, join our Gitter channel for all things Galaxy-single cell!
[![Gitter](https://badges.gitter.im/Galaxy-Training-Network/galaxy-single-cell.svg)](https://gitter.im/Galaxy-Training-Network/galaxy-single-cell?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge)
