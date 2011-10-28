LuxBlend DOF calculation addition for v071 and v08RC1
==============================

Addition for LuxBlend - DOF calculation for v071 and v08RC1 (Blender 2.49b). As I'm switching to 0.8RC1 (while still staying on 2.49b) added it to v08RC1 exporter version.

Calculator is shown if Use Depth of field is selected in Cam/Env tab and Focus type != autofocus
Although I always use f/ stop to define DOF, calculation retrieves value from lens-radius (luxprop of camera.lensradius). I believe I saw somewhere that DOF calculation inside Lux uses radius parameter. Anyway, this behaviour could be changed to use f/ number if selected.

Two block inserts:
* Class definition is on line 8705
* Instance is on 3314

Not evaluating close-up

Could add range of focus in front and behind the the focus point.

## DOF

```
#-------------------------------------------------
# dof calculator class
# does DOF calculation in exporter window, does not mess with other things :)
# kroko
#-------------------------------------------------
class cameraDofCalculator:
    def __init__(self, fLen, lensR, dofDist, coc = 0.03):
       # using mm internally
       # thus any instance getter method needs to be converted to m (blender units) in gui
       # default CoC value is for 35mm film
       self._fLen = fLen.getFloat() # mm
        self._lensR = lensR.getFloat()*1000 # m -> mm
        self._dofDist = dofDist.getFloat()*1000 #m -> mm
        self._coc = coc # mm
       if (self._fLen<=0.001 or self._lensR<=0.001 or self._dofDist<=0.001):
           self._valid = False
       else:
           self._valid = True
    # camera check
    def isValid(self):
        return self._valid                     
    # instance methods for DOF calculation
    # hyperfocal distance
    def getHyperFoc(self):
        return float((self._fLen/(self._coc/(2*self._lensR)))+self._fLen)
    # near distance
    def getDofNear(self):
        #distance * (hyperFocal - focal) / (hyperFocal + distance - 2*focal));
        return float(self._dofDist*(self.getHyperFoc()-self._fLen)/(self.getHyperFoc()+self._dofDist-2*self._fLen))
    # far distance
    def getDofFar(self):
       if (self._dofDist-self.getHyperFoc()>=0.0):
           return -1.0 # denotes infinity
       else:
           return float(self._dofDist*(self.getHyperFoc()-self._fLen)/(self.getHyperFoc()-self._dofDist))
    # total DOF
    def getDofField(self):
       if (self._dofDist-self.getHyperFoc()>=0.0):
           return -1.0 # denotes infinite
       else:
           return float(self.getDofFar()-self.getDofNear())
    # debug
    def getFLen(self):
       return float(self._fLen)
    def getLensR(self):
       return float(self._lensR)
    def getDofDist(self):
       return float(self._dofDist)


#-------------------------------------------------
# // end of dof calculator class
#-------------------------------------------------

```

## DOF

```
#-------------------------------------------------
# dof calculator instance
# kroko
#-------------------------------------------------
        
        if camtype.get() == "perspective" and usedof.get() == "true" and focustype.get() != "autofocus":
        
            # Input:
            # Focal length in mm
            # Lens radius (as the exporter converts f/ to radius anyway) in m
            # Focus distance in m
            # CoC in mm
            camDofCalculator = cameraDofCalculator(luxAttr(cam, "lens"), luxProp(cam, "camera.lensradius", 0.01), luxAttr(cam, "dofDist") )
        
            if gui: gui.newline("  DOF calc:")
            
            if camDofCalculator.isValid() == True:
                # Near distance
                r = gui.getRect(1,1); BGL.glRasterPos2i(r[0],r[1]+5)
                Draw.Text("Near distance: ", 'normal')
                r = gui.getRect(1,1); BGL.glRasterPos2i(r[0],r[1]+5)
                Draw.Text("%.3f m"%(camDofCalculator.getDofNear()/1000), 'normal')
                # Far distance
                r = gui.getRect(1,1); BGL.glRasterPos2i(r[0],r[1]+5)
                Draw.Text("Far distance: ", 'normal')
                r = gui.getRect(1,1); BGL.glRasterPos2i(r[0],r[1]+5)
                if (camDofCalculator.getDofFar() < 0):
                    Draw.Text("Infinity", 'normal')
                else:
                    Draw.Text("%.3f m"%(camDofCalculator.getDofFar()/1000), 'normal')
                # Depth of field
                r = gui.getRect(1,1); BGL.glRasterPos2i(r[0],r[1]+5)
                Draw.Text("Depth of field: ", 'normal')
                r = gui.getRect(1,1); BGL.glRasterPos2i(r[0],r[1]+5)
                if (camDofCalculator.getDofField() < 0):
                    Draw.Text("Infinite", 'normal')
                else:
                    Draw.Text("%.3f m"%(camDofCalculator.getDofField()/1000), 'normal')
                # Hyperfocal distance            
                r = gui.getRect(1,1); BGL.glRasterPos2i(r[0],r[1]+5)
                Draw.Text("Hyperfocal distance: ", 'normal')
                r = gui.getRect(1,1); BGL.glRasterPos2i(r[0],r[1]+5)
                Draw.Text("%.3f m"%(camDofCalculator.getHyperFoc()/1000), 'normal')
            else:
                r = gui.getRect(1,1); BGL.glRasterPos2i(r[0],r[1]+5)
                Draw.Text("Invalid camera parameters", 'normal')
                gui.newline()
                                     
            """
            # debug
            r = gui.getRect(1,1); BGL.glRasterPos2i(r[0],r[1]+5)
            Draw.Text("Focal length: ", 'normal')
            r = gui.getRect(1,1); BGL.glRasterPos2i(r[0],r[1]+5)
            Draw.Text("%.3f"%camDofCalculator.getFLen(), 'normal')
            
            r = gui.getRect(1,1); BGL.glRasterPos2i(r[0],r[1]+5)
            Draw.Text("Lens radius: ", 'normal')
            r = gui.getRect(1,1); BGL.glRasterPos2i(r[0],r[1]+5)
            Draw.Text("%.3f"%camDofCalculator.getLensR(), 'normal')
            
            r = gui.getRect(1,1); BGL.glRasterPos2i(r[0],r[1]+5)
            Draw.Text("DOF distance: ", 'normal')
            r = gui.getRect(1,1); BGL.glRasterPos2i(r[0],r[1]+5)
            Draw.Text("%.3f"%camDofCalculator.getDofDist(), 'normal')
            """
#-------------------------------------------------
# // end of dof calculator instance
#-------------------------------------------------

```

## CoC
Coc is assumed 0.03 mm (passed in init method as def value)
http://www.rags-int-inc.com/PhotoTechStuff/DoF/
http://www.dofmaster.com/digital_coc.html

```
            # Could retrieve aspect ratio of rendering, to make assumption about camera format and thus CoC
            # 
            # Different CoC may be used for DOF calculation, but it is irrelevant, as it does not glue with LuxRender "internal Coc"
            # i.e. in partly pseudocode
            #if gui:
            #    # Toogle to set if preset should be used or CoC set manually
            #    r = gui.getRect(1.0, 1.0)
            #    Draw.Toggle("Use CoC preset", anEvent, r[0], r[1], r[2], r[3], 1,  "Use CoC preset for a format")
            #    if toogle activated:
            #        # Choose a film/sensor (aspect ratio given), each defined its CoC (using values from tables found on web)
            #        r = gui.getRect(1.0, 1.0)
            #        name = "Camera format %t|35mm film / full frame DSLR (1.5) |APS-C DSLR (1.5) |Four Thirds DSLR (1.3(3)) |645 (1.3(3)) %x4|6x6 (1) |6x7 (1.16(6))"
            #        Draw.Menu(name...)
            #    else:
            #        # Number field to enter CoC manually
            #        r = gui.getRect(1.0, 1.0)
            #        Draw.Number("CoC", anEvent, r[0], r[1], r[2], r[3], 5, 1, 10, "CoC value in mm")

"""
35mm film -  full frame DSLR - (3:2)
http://en.wikipedia.org/wiki/Full-frame_digital_SLR
http://en.wikipedia.org/wiki/35_mm_film
APS-C DSLR (crop factor) (3:2)
http://en.wikipedia.org/wiki/APS-C
Four Thirds DSLR (4:3)
http://en.wikipedia.org/wiki/Four_Thirds_system
645
6x6
6x7
"""

```


See LuxBlend_0.1.py