## Project: 3D Motion Planning



## [Rubric](https://review.udacity.com/#!/rubrics/1534/view) Points
### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---
### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  

You're reading it! Below I describe how I addressed each rubric point and where in my code each point is handled.

### Explain the Starter Code

#### 1. Explain the functionality of what's provided in `motion_planning.py` and `planning_utils.py`

**About `motion_planning.py`**

Like in the `backyardflyer` project I have completed the `motion_planning.py`  contains the main code that will plan run the commmands to guide the drone. The programming paradigm in which the program is written doesn't rely on time to schedule the path of the drone but on event programming. There are several distinct events like "Take Off", "Landing", "Arming", etc. The program takes care the sequence of these events so that the it is still responsive if ,while it's moving, there's an unexpected obstacle. Also, the program is responsible for calling the planning function so that it can find a path between a starting and a goal location. Finding a path is done using a modified version of the `a_star` algorithm that it is defined in `planning_utils.py`. I have also included diagonal movements beyond what was provided in the starter code as this was a requirement (method `valid_actions()`.

### Implementing Your Path Planning Algorithm

#### 1. Set your global home position

Starting the motion planner program first saves the central location as:
`
lat0,lon0
37.792480,-122.397450
`
In which it is then decoded in the in a local location. This is done in the following part:

```
        with open("colliders.csv") as myfile:
            head = [next(myfile) for x in range(2)]
        latlon = np.fromstring(head[1], dtype='Float64', sep=',')
        lat0 = latlon[0]
        lon0 = latlon[1]
        # TODO: set home position to (lat0, lon0, 0)
        self.set_home_position(lon0, lat0, 0)
```

I read the csv file twice because the file includes two formats (different columnson each one).


This is where the drone starts in the simulator:
![Map of SF](start.png)

#### 2. Set your current local position

I set the local position relative to the global home position using the following line:

```
current_local_pos = global_to_local(self.global_position,self.global_home)
```

I have previously set the home position in the line:

```
self.set_home_position(lon0, lat0, 0)
```
The lan0 and lat0 where retrieved from the `.csv`.


#### 3. Set grid start position from local position
This is another step in adding flexibility to the start location. As long as it works you're good to go!

I set the grid start position in the line:

```
start = (int(current_local_pos[0]+north_offset), int(current_local_pos[1]+east_offset))
```

I have taken into account the north at east offset on the map to find the place in the grid.

#### 4. Set grid goal position from geodetic coords

The goal is set using the two lines:

```
grid_goal = global_to_local((-122.401247,37.796738,0),self.global_home)
grid_goal = (int(grid_goal[0]+ north_offset),int(grid_goal[1]+ east_offset))
```

As you can see the input is geodetic coordinates `(-122.401247,37.796738,0)` from which I retrieve the local coordinates using `global_to_local`. The user can also select their goal from running the script with:


```
python motion_planning.py --lat 37.796738 --lon -122.401247
```

This was done by adding two arguments:
```
    parser.add_argument('--lat', type=float, default=1000, help="latitude")
    parser.add_argument('--lon', type=float, default=1000, help="latitude")
```

#### 5. Modify A* to include diagonal motion (or replace A* altogether)

I have modified the selection of next moves in the A* to include diagonal motions:

The actions includes four new ones (diagonal) with cost `sqrt(2)`:
```
    NORTHWEST = (-1, -1, 1.41421)
    SOUTHWEST = (1, -1, 1.41421)
    NORTHEAST = (-1,1,1.41421)
    SOUTHEAST = (1,1,1.41421)
```

and in the `valid_actions` I have added:

```
    if x - 1 < 0 or y - 1 < 0 or grid[x - 1, y - 1] == 1:
        valid_actions.remove(Action.NORTHWEST)
    if x + 1 > n or y - 1 < 0 or grid[x + 1, y - 1] == 1:
        valid_actions.remove(Action.SOUTHWEST)
    if x - 1 < 0 or y + 1 > m or grid[x - 1, y + 1] == 1:
        valid_actions.remove(Action.NORTHEAST)
    if x + 1 > n or y + 1 > m or grid[x + 1, y + 1] == 1:
        valid_actions.remove(Action.SOUTHEAST)
```


#### 6. Cull waypoints

I used collinearity to prune the path. The prunning algorithm looks like this:

```
    def prune_path(self,path):
        def point(p):
            return np.array([p[0], p[1], 1.]).reshape(1, -1)

        def collinearity_check(p1, p2, p3, epsilon=1e-6):
            m = np.concatenate((p1, p2, p3), 0)
            det = np.linalg.det(m)
            return abs(det) < epsilon
            
        pruned_path = []
        # TODO: prune the path!
        p1 = path[0]
        p2 = path[1]
        pruned_path.append(p1)
        for i in range(2,len(path)):
            p3 = path[i]
            if collinearity_check(point(p1),point(p2),point(p3)):
                p2 = p3
            else:
                pruned_path.append(p2)
                p1 = p2
                p2 = p3
        pruned_path.append(p3)


        return pruned_path
```

In collinearity I select continuous groups of points and (3) to see if the belong in a line or close to a line. If they can be connected to a line I replace the two waypoints with a single on (longer) and continue the search to see if I can add more way points to the same line. With this change I managed to have a relatively smooth route:

![img](path.png)





### Execute the flight
#### 1. Does it work?
I tried the suggested location of ` (longitude = -122.402224, latitude = 37.797330)` and the drone guided itself into it. To go back just run from there:

```
python motion_planning.py --lat 37.792480 --lon -122.397450
```

You can run any location you like from there using the parameters `lat,lon`.

### Extra Challenges: Real World Planning 

I did not yet attempt any challenges but I plan to do later in the course.

For an extra challenge, consider implementing some of the techniques described in the "Real World Planning" lesson. You could try implementing a vehicle model to take dynamic constraints into account, or implement a replanning method to invoke if you get off course or encounter unexpected obstacles.