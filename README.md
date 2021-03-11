# Dorado observation planning and scheduling simulations

Dorado is a proposed space mission for ultraviolet follow-up of gravitational
wave events. This repository contains a simple target of opportunity
observation planner for Dorado.

![Example Dorado observing plan](examples/6.gif)

## Features

*   **Global**: jointly and globally solves the problems of tiling (the set of
    telescope boresight orientations and roll angles) and the scheduling (which
    tile is observed at what time), rather than solving each sub-problem one at
    a time
*   **Optimal**: generally solves all the way to optimality, rather than
    finding merely a "good enough" solution
*   **Fast**: solve an entire orbit in about 5 minutes
*   **General**: does not depend on heuristics of any kind
*   **Flexible**: problem is formulated in the versatile framework of
    [mixed integer programming]

## Dependencies

*   [Astropy]
*   [Astroplan] for calculating the field of regard
*   [HEALPix], [Healpy], and [astropy-healpix] for observation footprints
*   [Skyfield] for orbit propagation
*   [CPLEX] (via [docplex] Python interface) for constrained optimization

## Problem formulation

Given a gravitational-wave HEALPix probability sky map, this Python package
finds an optimal sequence of Dorado observations to maximize the probability of
observing the (unknown) location of the gravitational-wave event, within one
orbit.

The problem is formulated as a mixed integer programming problem with the
following arrays of binary decision variables:

*   `schedule` (`npix` × `nrolls` × `ntimes - ntimes_per_exposure + 1`): 1 if
    an observation of the field that is centered on the given HEALPix pixel, at
    a given roll angle, is begun on a given time step; or 0 otherwise
*   `pix` (`npix`): 1 if the given HEALPix pixel is observed, or 0
    otherwise

The problem has the following parameters:

*   `nexp` (scalar, integer): the maximum number of exposures
*   `prob` (`npix`, float): the probability sky map
*   `regard` (`npix` × `ntimes`, binary): 1 if the field centered on
    the given HEALPix pixel is within the field of regard at the given time, 0
    otherwise

The objective function is the sum over all of the pixels in the probability sky
map for which the corresponding entry in `pix` is 1.

The constraints are:
*   At most one observation is allowed at a time.
*   At most `nexp` observations are allowed in total.
*   A given pixel is observed if any field that contains it within its
    footprint is observed.
*   A field may be observed only if it is within the field of regard.

## Usage

### To install

To install with [Pip]:

1.  Run the following command:

        $ pip install git+https://github.com/dorado-science/dorado-scheduling

### To set up the CPLEX optimization engine

2.  Set up the CPLEX optimization engine by following the
    [docplex instructions]. If you have [installed CPLEX locally], then all you
    have to do is determine the path to the CPLEX Python bindings and add them
    to your `PYTHONPATH`. For example, on macOS, this might be:

        $ export PYTHONPATH=/Applications/CPLEX_Studio1210/cplex/python/3.7/x86-64_osx

### To generate an observing plan

3.  Generate an observing plan for the included example sky map:

        $ dorado-scheduling examples/6.fits -o examples/6.ecsv

    This will take 3 minutes and will use about 10 GB of memory at peak.

4.  Print out the observing plan:

        $ cat examples/6.ecsv 
        # %ECSV 0.9
        # ---
        # datatype:
        # - {name: time, datatype: string}
        # - {name: center.ra, unit: deg, datatype: float64}
        # - {name: center.dec, unit: deg, datatype: float64}
        # - {name: roll, unit: deg, datatype: float64}
        # meta:
        #   __serialized_columns__:
        #     center:
        #       __class__: astropy.coordinates.sky_coordinate.SkyCoord
        #       dec: !astropy.table.SerializedColumn
        #         __class__: astropy.coordinates.angles.Latitude
        #         unit: &id001 !astropy.units.Unit {unit: deg}
        #         value: !astropy.table.SerializedColumn {name: center.dec}
        #       frame: icrs
        #       ra: !astropy.table.SerializedColumn
        #         __class__: astropy.coordinates.angles.Longitude
        #         unit: *id001
        #         value: !astropy.table.SerializedColumn {name: center.ra}
        #         wrap_angle: !astropy.coordinates.Angle
        #           unit: *id001
        #           value: 360.0
        #       representation_type: spherical
        #     time:
        #       __class__: astropy.time.core.Time
        #       format: isot
        #       in_subfmt: '*'
        #       out_subfmt: '*'
        #       precision: 3
        #       scale: utc
        #       value: !astropy.table.SerializedColumn {name: time}
        #   cmdline: dorado-scheduling examples/6.fits -o examples/6.ecsv
        #   prob: 0.8238346278346581
        #   status: OPTIMAL
        # schema: astropy-2.0
        time center.ra center.dec roll
        2012-05-02T18:29:32.699 54.47368421052631 -61.94383702315671 80.0
        2012-05-02T18:44:32.699 69.75 -60.434438844952275 50.0
        2012-05-02T19:32:32.699 147.65625 -7.180755781458282 70.0
        2012-05-02T19:42:32.699 115.31249999999999 18.20995686428301 20.0
        2012-05-02T19:56:32.699 133.59375 7.180755781458282 20.0

5.  To generate an animated visualization for this observing plan, run the
    following command:

        $ dorado-scheduling-animate examples/6.fits examples/6.ecsv examples/6.gif

    This will take about 2-5 minutes to run.

### Determining if a given sky position is contained within an observing plan

The following example illustrates how to use HEALPix to determine if a given
sky position is contained in any of the fields in an observing plan:

```pycon
>>> from astropy.coordinates import SkyCoord
>>> from astropy.table import QTable
>>> from astropy import units as u
>>> from dorado.scheduling import skygrid
>>> target = SkyCoord(66.91436579*u.deg, -61.98378895*u.deg)
>>> target_pixel = skygrid.healpix.skycoord_to_healpix(target)
>>> schedule = QTable.read('examples/6.ecsv')
>>> footprints = [skygrid.get_footprint_healpix(row['center'], row['roll'])
...               for row in schedule]
>>> schedule['found'] = [target_pixel in footprint for footprint in footprints]
>>> schedule
<QTable length=5>
          time                         center                  roll  found
                                      deg,deg                  deg
         object                        object                float64  bool
----------------------- ------------------------------------ ------- -----
2012-05-02T18:29:32.699 54.47368421052631,-61.94383702315671    80.0 False
2012-05-02T18:44:32.699            69.75,-60.434438844952275    50.0  True
2012-05-02T19:32:32.699         147.65625,-7.180755781458282    70.0 False
2012-05-02T19:42:32.699 115.31249999999999,18.20995686428301    20.0 False
2012-05-02T19:56:32.699          133.59375,7.180755781458282    20.0 False
```

[Pip]: https://pip.pypa.io
[mixed integer programming]: https://en.wikipedia.org/wiki/Integer_programming
[Astropy]: https://www.astropy.org
[Astroplan]: https://github.com/astropy/astroplan
[HEALPix]: https://healpix.jpl.nasa.gov
[astropy-healpix]: https://github.com/astropy/astropy-healpix
[Healpy]: https://github.com/healpy/healpy
[Skyfield]: https://rhodesmill.org/skyfield/
[install Poetry]: https://python-poetry.org/docs/#installation
[CPLEX]: https://www.ibm.com/products/ilog-cplex-optimization-studio
[docplex]: https://ibmdecisionoptimization.github.io/docplex-doc/
[docplex instructions]: https://ibmdecisionoptimization.github.io/docplex-doc/mp/getting_started.html
[installed CPLEX locally]: https://ibmdecisionoptimization.github.io/docplex-doc/mp/getting_started.html#using-ibm-ilog-cplex-optimization-studio-on-your-computer
