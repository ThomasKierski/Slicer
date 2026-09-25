## Plots

### Slicer plots displayed in view layout

Create histogram plot of a volume and show it embedded in the view layout. More information: https://www.slicer.org/wiki/Documentation/Nightly/Developers/Plots

### Using {func}`slicer.util.plot()` utility function

```python
# Get a volume from SampleData and compute its histogram
import SampleData
import numpy as np
volumeNode = SampleData.SampleDataLogic().downloadMRHead()
histogram = np.histogram(slicer.util.arrayFromVolume(volumeNode), bins=50)

chartNode = slicer.util.plot(histogram, xColumnIndex = 1)
chartNode.SetYAxisRangeAuto(False)
chartNode.SetYAxisRange(0, 4e5)
```

![Plot displayed using Slicer's plotting module](https://www.slicer.org/w/img_auth.php/9/9c/SlicerPlot.png)

#### Using MRML classes only

```python
# Get a volume from SampleData
import SampleData
volumeNode = SampleData.SampleDataLogic().downloadMRHead()

# Compute histogram values
import numpy as np
histogram = np.histogram(slicer.util.arrayFromVolume(volumeNode), bins=50)

# Save results to a new table node
tableNode=slicer.mrmlScene.AddNewNodeByClass("vtkMRMLTableNode")
slicer.util.updateTableFromArray(tableNode, histogram)
tableNode.GetTable().GetColumn(0).SetName("Count")
tableNode.GetTable().GetColumn(1).SetName("Intensity")

# Create plot
plotSeriesNode = slicer.mrmlScene.AddNewNodeByClass("vtkMRMLPlotSeriesNode", volumeNode.GetName() + " histogram")
plotSeriesNode.SetAndObserveTableNodeID(tableNode.GetID())
plotSeriesNode.SetXColumnName("Intensity")
plotSeriesNode.SetYColumnName("Count")
plotSeriesNode.SetPlotType(plotSeriesNode.PlotTypeScatterBar)
plotSeriesNode.SetColor(0, 0.6, 1.0)

# Create chart and add plot
plotChartNode = slicer.mrmlScene.AddNewNodeByClass("vtkMRMLPlotChartNode")
plotChartNode.AddAndObservePlotSeriesNodeID(plotSeriesNode.GetID())
plotChartNode.YAxisRangeAutoOff()
plotChartNode.SetYAxisRange(0, 500000)

# Show plot in layout
slicer.modules.plots.logic().ShowChartInLayout(plotChartNode)
```

### Save a plot as vector graphics (.svg)

```python
plotView = slicer.app.layoutManager().plotWidget(0).plotView()
plotView.saveAsSVG("c:/tmp/test.svg")
```

### Using matplotlib

Matplotlib may be used from within Slicer. Slicer is built without Tcl/Tk, so the default
`TkAgg` backend is unavailable, and Slicer binds Qt through
[PythonQt](https://mevislab.github.io/pythonqt/) rather than PyQt or PySide, so
Matplotlib's own `QtAgg` backend cannot be used either. Do not install PyQt or PySide into
Slicer's Python to work around this: doing so loads a second, independently initialized
copy of the Qt libraries into a process that has already initialized Slicer's own Qt,
which is unstable.

Instead, Slicer provides a built-in interactive backend, `slicer.matplotlibbackend`, which
renders with Agg and displays the result in a Qt widget driven by Slicer's own event loop.
Use `Agg` when you only need to render an image file. More details can be found on the
[MatPlotLib](https://matplotlib.org/) pages.

#### Interactive plot

```python
try:
  import matplotlib
except ModuleNotFoundError:
  slicer.packaging.pip_install("matplotlib")
  import matplotlib

# Select Slicer's interactive backend and enable interactive mode.
import slicer.matplotlibbackend
slicer.matplotlibbackend.enable()

# Get a volume from SampleData and compute its histogram
import SampleData
import numpy as np
volumeNode = SampleData.SampleDataLogic().downloadMRHead()
histogram = np.histogram(slicer.util.arrayFromVolume(volumeNode), bins=50)

# Show an interactive plot: pan, zoom, and the navigation toolbar all work.
import matplotlib.pyplot as plt
fig, ax = plt.subplots()
ax.plot(histogram[1][1:], histogram[0].astype(float))
ax.grid(True)
ax.set_ylim((0, 4e5))
plt.show()
```

Because Slicer's application event loop is always running, `plt.show()` returns
immediately and never blocks the application.

#### Embed a plot in a module panel

The canvas is an ordinary Qt widget, so it can be added to any layout, for example in the
`setup()` method of a scripted module:

```python
from matplotlib.figure import Figure
from slicer.matplotlibbackend import FigureCanvasSlicer, NavigationToolbar2Slicer

figure = Figure()
ax = figure.add_subplot(111)
ax.plot([0, 1, 2], [0, 1, 0])

canvas = FigureCanvasSlicer(figure)
toolbar = NavigationToolbar2Slicer(canvas)

self.layout.addWidget(canvas.get_widget())
self.layout.addWidget(toolbar.get_widget())
```

Keep a reference to `canvas` (for example on `self`) for as long as the plot is displayed.
Call `canvas.draw_idle()` after modifying the figure to schedule a repaint.

By default the canvas reports the figure size as its preferred size, so it claims a large
share of a shared layout. Call `canvas.set_size_hint(width, height)` with a small size to
let the surrounding layout drive the geometry instead, or `canvas.set_size_hint()` to
restore the default.

#### Interactive plot in the Four-Up Plot layout

This example replaces the VTK plot view of the `Four-Up Plot` layout with an interactive
Matplotlib figure, and wires it to the scene in both directions:

- scrolling the red slice view updates the histogram of the slice that is actually displayed,
- dragging a range over the histogram applies it as the window/level of the volume.

```python
import numpy as np
import vtk
from vtk.util import numpy_support
import SampleData

try:
  import matplotlib
except ModuleNotFoundError:
  slicer.packaging.pip_install("matplotlib")
  import matplotlib

import slicer.matplotlibbackend
slicer.matplotlibbackend.enable()

from matplotlib.figure import Figure
from matplotlib.widgets import SpanSelector
from slicer.matplotlibbackend import FigureCanvasSlicer, NavigationToolbar2Slicer


class SliceHistogramPlot:
    """Interactive Matplotlib histogram of the slice currently shown in a slice view."""

    def __init__(self, volumeNode, sliceViewName="Red"):
        self.volumeNode = volumeNode
        self.sliceLogic = slicer.app.layoutManager().sliceWidget(sliceViewName).sliceLogic()
        self.sliceNode = self.sliceLogic.GetSliceNode()

        self.figure = Figure(tight_layout=True)
        self.axes = self.figure.add_subplot(111)
        self.canvas = FigureCanvasSlicer(self.figure)
        self.toolbar = NavigationToolbar2Slicer(self.canvas)
        # Let the Slicer layout drive the size instead of the figure size.
        self.canvas.set_size_hint(100, 100)

        # Drag over the histogram to apply that intensity range as window/level.
        self.spanSelector = SpanSelector(
            self.axes, self.onIntensityRangeSelected, "horizontal",
            useblit=True, props=dict(alpha=0.3, facecolor="tab:orange"),
            interactive=True, drag_from_anywhere=True)

        self.sliceObserver = self.sliceNode.AddObserver(
            vtk.vtkCommand.ModifiedEvent, self.onSliceModified)
        self.update()

    def addToPlotView(self):
        """Replace the VTK plot view of the Four-Up Plot layout with this canvas."""
        plotWidget = slicer.app.layoutManager().plotWidget(0)
        plotWidget.plotView().hide()
        plotWidget.layout().addWidget(self.canvas.get_widget())
        plotWidget.layout().addWidget(self.toolbar.get_widget())

    def removeFromPlotView(self):
        self.sliceNode.RemoveObserver(self.sliceObserver)
        plotWidget = slicer.app.layoutManager().plotWidget(0)
        self.canvas.get_widget().setParent(None)
        self.toolbar.get_widget().setParent(None)
        plotWidget.plotView().show()

    def currentSliceArray(self):
        """Voxels of the reslice actually displayed, for any slice orientation."""
        reslice = self.sliceLogic.GetBackgroundLayer().GetReslice()
        reslice.Update()
        scalars = reslice.GetOutput().GetPointData().GetScalars()
        if scalars is None:
            return None
        return numpy_support.vtk_to_numpy(scalars)

    def onSliceModified(self, caller=None, event=None):
        self.update()

    def onIntensityRangeSelected(self, minIntensity, maxIntensity):
        if maxIntensity <= minIntensity:
            return
        displayNode = self.volumeNode.GetDisplayNode()
        displayNode.AutoWindowLevelOff()
        displayNode.SetWindowLevelMinMax(minIntensity, maxIntensity)

    def update(self):
        voxels = self.currentSliceArray()
        self.axes.clear()
        if voxels is not None and voxels.size:
            # Ignore the zero-valued background that reslicing introduces.
            voxels = voxels[voxels > 0]
        if voxels is not None and voxels.size:
            self.axes.hist(voxels, bins=80, color="tab:blue")
        self.axes.set_xlabel("Intensity")
        self.axes.set_ylabel("Voxel count")
        self.axes.set_title("Slice offset %.1f mm - drag to set window/level"
                            % self.sliceNode.GetSliceOffset())
        self.axes.grid(True, alpha=0.3)
        self.canvas.draw_idle()


layoutManager = slicer.app.layoutManager()
layoutManager.setLayout(slicer.vtkMRMLLayoutNode.SlicerLayoutFourUpPlotView)

volumeNode = SampleData.SampleDataLogic().downloadMRHead()
slicer.util.setSliceViewerLayers(background=volumeNode, fit=True)

# Keep a reference so that the object and its observers stay alive.
slicer.modules.SliceHistogramPlotDemo = SliceHistogramPlot(volumeNode, "Red")
slicer.modules.SliceHistogramPlotDemo.addToPlotView()
```

Call `slicer.modules.SliceHistogramPlotDemo.removeFromPlotView()` to remove the observer
and restore the regular VTK plot view.

#### Non-interactive plot

Use the `Agg` backend to render a figure to an image file without showing a window:

```python
try:
  import matplotlib
except ModuleNotFoundError:
  slicer.packaging.pip_install("matplotlib")
  import matplotlib

matplotlib.use("Agg")
from pylab import *

t1 = arange(0.0, 5.0, 0.1)
t2 = arange(0.0, 5.0, 0.02)
t3 = arange(0.0, 2.0, 0.01)

subplot(211)
plot(t1, cos(2*pi*t1)*exp(-t1), "bo", t2, cos(2*pi*t2)*exp(-t2), "k")
grid(True)
title("A tale of 2 subplots")
ylabel("Damped")

subplot(212)
plot(t3, cos(2*pi*t3), "r--")
grid(True)
xlabel("time (s)")
ylabel("Undamped")
savefig("MatplotlibExample.png")

# Static image view
pm = qt.QPixmap("MatplotlibExample.png")
imageWidget = qt.QLabel()
imageWidget.setPixmap(pm)
imageWidget.setScaledContents(True)
imageWidget.show()
```

:::{tip}
To learn how to use {func}`slicer.packaging.pip_install` within a Slicer module, refer to the [](/developer_guide/script_repository.md#install-a-python-package) example in the Script Repository.
:::

![Matplotlib example](https://www.slicer.org/w/img_auth.php/a/ab/MatplotlibExample.png)

#### Plot in Slicer Jupyter notebook

```python
import JupyterNotebooksLib as slicernb
try:
  import matplotlib
except ModuleNotFoundError:
  slicer.packaging.pip_install("matplotlib")
  import matplotlib

matplotlib.use("Agg")

import matplotlib.pyplot as plt
import numpy as np

def f(t):
  s1 = np.cos(2*np.pi*t)
  e1 = np.exp(-t)
  return s1 * e1

t1 = np.arange(0.0, 5.0, 0.1)
t2 = np.arange(0.0, 5.0, 0.02)
t3 = np.arange(0.0, 2.0, 0.01)


fig, axs = plt.subplots(2, 1, constrained_layout=True)
axs[0].plot(t1, f(t1), "o", t2, f(t2), "-")
axs[0].set_title("subplot 1")
axs[0].set_xlabel("distance (m)")
axs[0].set_ylabel("Damped oscillation")
fig.suptitle("This is a somewhat long figure title", fontsize=16)

axs[1].plot(t3, np.cos(2*np.pi*t3), "--")
axs[1].set_xlabel("time (s)")
axs[1].set_title("subplot 2")
axs[1].set_ylabel("Undamped")

slicernb.MatplotlibDisplay(matplotlib.pyplot)
```

![Example for using Matplotlib in a Slicer Jupyter Notebook](https://www.slicer.org/w/img_auth.php/a/a2/JupyterNotebookMatplotlibExample.png)


#### Interactive plot using wxWidgets GUI toolkit

:::{note}
This approach predates the built-in `slicer.matplotlibbackend` described above, which is
now the recommended way to create interactive plots. `WXAgg` requires the additional
wxPython dependency and can only show figures in separate top-level windows.
:::

```python
try:
  import matplotlib
  import wx
except ModuleNotFoundError:
  slicer.packaging.pip_install("matplotlib wxPython")
  import matplotlib

# Get a volume from SampleData and compute its histogram
import SampleData
import numpy as np
volumeNode = SampleData.SampleDataLogic().downloadMRHead()
histogram = np.histogram(slicer.util.arrayFromVolume(volumeNode), bins=50)

# Set matplotlib to use WXAgg backend
import matplotlib
matplotlib.use("WXAgg")

# Show an interactive plot
import matplotlib.pyplot as plt
fig, ax = plt.subplots()
ax.plot(histogram[1][1:], histogram[0].astype(float))
ax.grid(True)
ax.set_ylim((0, 4e5))
plt.show(block=False)
```

:::{tip}
To learn how to use {func}`slicer.packaging.pip_install` within a Slicer module, refer to the [](/developer_guide/script_repository.md#install-a-python-package) example in the Script Repository.
:::

![Interactive Matplotlib Example](https://www.slicer.org/w/img_auth.php/d/d2/InteractiveMatplotlibExample.png)
