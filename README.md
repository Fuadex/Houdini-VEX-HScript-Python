# Houdini VEX/Hscript
 Collection of scripts I've developed

Houdini VEX and a bit of Hscript for now.

-----------------------------------------------

CharFX Preroll Fix

I came across a few instances while working on cloth simulations, when the preroll didn't exactly end up smoothly at the end of the preroll. What's preroll? Well, for any simulated asset it's extra frames which may range between 15-30-50-100 frames so that once shot goes in motion, it's already settled or continuing from previous shot.
Anyway, it's a bit of a process setting up metadata, getting the animator to publish a new cache and this stuff is fairly automatized anyway in most circumstances but sometimes you still end up getting a bit of a blank, that kind of screams for a kickback to anim.
I thought that, maybe I could actually do my own preroll and I sorta ended up getting the exact smooth transition but also I managed to optimize simulation time at once. How?

Chuck down a timeshift set to beginning of shot (preroll) and another with actual beginning of the shot. Plug both into a point wrangle.

	vector P1 = point(0,"P",@ptnum);
	vector P2 = point(1,"P",@ptnum);
	int T1 = $RFSTART;
	int T2 = $RFSTART + 100;
	float shot_length = fit(@Frame,T1,T2,0,1);
	float ramp = chramp("Translate",shot_length);

	v@P = P1+(P1-P2)*ramp*-1;

Set the ramp to have a smooth transition, with B-Spline or whatever.
Incoming anim geo can end up being heavy and having it load frame by frame, unpacked, converted for collisions etc. can siphon quite a bit of time.
This kinda hunts down two birds with one stone.

As an extra thing - Vellum ramping

I had a problem that I couldn't crack easily. I mean, remapping values is cool and all until you need to remap vellum values at dop level and it's stiffness stuff. I couldn't exactly comprehend why my maths were failing or was it my head that was failing or is everything failing? etc. etc.
Well, turns out that, duh, obviously, stiffness stuff in vellum operates in exponential values of 10 to the power of whatevs. So really, remapping should function in relation to the exponent itself.

So....? Say, you want to remap stiffness properly between specific frames, here's what you need and best sourced outside of dops. For sanity reasons.

	i@frame_start = $FSTART;
	i@frame_end = $FEND;
	f@stiffness = 1;
	f@exponent = fit(@Frame,i@frame_start,i@frame_end,-10,10);

	f@remap = 1 * pow(10,@exponent);

Now this will remap stiffness the way you'd think it logically should. And if you're after some smooth stiffness increase/decrease that just simply looks good, this is the stuff.

--------------------------------------------------------

CharFX vector deformation and Bend node

This one I thought of waaay too late and should've come up with this a long time ago.
Say, you've got cloth geo and a collider anim and anim is at rest but collides initially with cloth. What do you do?
Well, most of the time I'd peak in the anim geo and then slowly animate it outwards but you can't always do this as collision geo can have have extrusions or crawl up into itself.

I thought of a very simple solution really - just split some part of the geo you're interested in, can be procedural or not you decide - extract centroid
Now into a pointwrangle

	//vector P1 = point(0,"P",0);
	vector P2 = point(1,"P",0);
	float min = chf("minimum");
	float max = chf("maximum");

	float ratio = chf("ratio");
	float dist = distance(v@P,P2);
	float remap = fit(dist,min,max,1,0);
	//float bend = chramp("bend",remap);

	v@P = v@P + (v@P-P2)*-(ratio)*remap; //use bend instead of remap!

So, you're probably wondering what does this do - best I can describe it, it's like a metaball - you pull all the points towards a specific direction
Simple script as hell, you can add more points, maybe not have one point but a point cloud instead, whatever - but heck, vector maths are king.

You know what's also an option? I had a problem with which using a Bend node was a no-no due to the thing stretching in mysterious ways. So what did my little brain think of? Ha, custom bend node! I mean it's essentially the same thing but... include a chramp based on the remapped value and you can also create your own vector. I put it commented out if you'd try to take it for a spin yourself.

----------------------------------------------------------

Wrangle blendshape

This one's easy - I mean it's just a blendshape but I always hated the idea of trying to recreate this, so I just did once after leaving Fin because I knew if I didn't try recreating straight afterwards I'd probably end up spending way too much time doing this after completing my break.

	vector P1 = point(1,"P",@ptnum);
	vector P2 = point(0,"P",@ptnum);
	vector deltaP = P2 - P1;

	float offset=ch("offset");
	float foo = clamp((f@float_variable+offset),-1,1);
	float scale = chramp("ramppos",foo);

	v@P = P1 + (scale*deltaP);

I mean, again - it's just a blendshape, as long as point count is the same - you can just blend all the points and actually do this based on a mask of your choosing, which actually it's reallly powerful.

-------------------------------------------------------

Vector push and direction estimation

Very simple push by a vector direction

	vector dir1 = set(0,0,0);
	vector dir2 = set(1,1,1);
	v@P += dir2 * chf("magnitude");

It just moves man. That's for a pointwrangle. But actually, maybe we can do this more efficiently, huh?

Detail wrangle version. To be frank, I'm not entirely sure what I use this one for but I'll get back to ya soon, k?

	vector dir;

	for(int=0; i<nprimitives(0); i++){
		dir += prim(0,"N",i);
		dir = normalize(dir);
	}

Calculate vector

This one I used to estimate the direction of a curve. I used this to variably switch point and uv count on some curves back in the CFX days.
I'm sure there's a neater trick but as a quick and dirty method, even if the curve goes up and down and sideways, it should work.

Chuck down a Resample node, get the tangents then detail wrangle

	v[]@tangentu_list;
	v@vector_sum;

	for(int i=0; i<@numpt; i++){
		vector tangentu_pt = point(0,"tangentu",i);
		append(@tangentu_list, tangentu_pt);
	}

	@vector_sum = sum(@tangentu_list)/len(@tangentu_list);


-------------------------------------------------------

In Relation Operations

Once upon a time I believed in relpointbbox. For instance, if I wanted to vary up attribs based on height, like with this

	vector relative = relpointbbox(0,@P);
	float multiplier = chf("multiplier");

	f@Relative_y = chramp("Height",relative[1])*multiplier;

Yup, that works but bbox was always a bit of a weird pain for me. When watching Junichiro Horikawa's VEX series, I go into intrinsics and hey, I luv em!

It's simple.

	float bounds[] = detailintrinsic(0,"bounds");
	float bounds_ymin = bounds[2];
	float bounds_ymax = bounds[3];

	//And you do logic.
	//Such as this.

	vector spread = fit(v@P, bounds_ymin, bounds_ymax,0,1);
	float ramp = chramp("ramp",spread[1]);
	@Cd.r = ramp; //color red along the y axis of the object's bounding box

It's really cool beans. So handy. And there are intrinsics like that for primitives as well!


--------------------------------------------------------

Calculate radius

Boooring. But was still cool to give this a crack as I had a task in which I had to calculate radius of every single wire. Won't work for everything but when you need stuff calculated per object, it could work nicely in a detail wrangle and in a node-based for loop from connectivity. Ideally I'd spice this up to make sure it always worked but you can't always negotiate what sort of geo you'll be getting as a present today.

	float length[];

	for(int i=0;i<@numpt-1;i++){
		vector pt1 = point(0,"P",i);
		vector pt2 = point(0,"P",i+1);
		float measure = distance(pt1,pt2);
		push(length,measure);
	}

	vector pt1 = point(0,"P",0);
	vector pt2 = point(0,"P",@numpt-1);
	float measure = distance(pt1,pt2);
	push(length,measure);

	f@radius = ((sum(length))/(2*$PI))+0.001;

-------------------------------------------------------

Find matchmaker

Sometimes, I had issues, needing to select multiple things in code but not being able to do it in a neat way.
Here's a few cool approaches

	i[]@last = primpoints(0,@primnum)[-2:];
	i[]@first = primpoints(0,@primnum)[:2];

	if(find(i@last,@ptnum)==0 || find(i@last,@ptnum)==1){
	        v@Cd = 0;
	}

I also liked using match() in pointwrangles to get stuff by using wildcards, so example:

	if(match("*some_path*",prim(0,"path",@primnum)==1)){
		//do something
	}

If you can get stuff in a conditional statement that is readable, everyone wins

-------------------------------------------------------

Custom deformer

Say you're in a situation in which a shot begins with a character colliding with another object. You can get some stuff right with preroll but when your thing begins from already colliding with another thing, what can you do?
Well, took me a while to crack this and this was still not an ideal solution but it's a cool solution nevertheless. Whenever I can, I try going at problems procedurally but sometimes you need to work with a previous frame.
So... solver time! And time to crack your head trying to figure out in your brain how to solve a problem. Here's a solution it took me about 2 hours to crack.

Have a wrangle. Put a switch upstream to use first input at first frame and moving forward the previous frame.
Make sure you plug in your first input to keep track of the changing points position.
And plug in through another input the changing velocity computed from upstream.
Also do some vdb magic with your collider to source the sweet sdf goodness.

	float offset = chf("offset");
	vector sample = @P + @N * offset;
	float dist = volumesample(1,0,@P);
	vector distgradient = volumegradient(1,0,@P);


	f@intersect = 1;
	float multiply = chf("multiply");
	@P = (@P + (distgradient * multiply));
	v@velocity = point(3,"v",@ptnum);
	v@speed = point(3,"speed",@ptnum);

	@P = @P + (normalize(v@velocity)) * v@speed/24;

What essentially I'm doing, is pushing out the collider to make space for whatever is about to start colliding with. It's a custom deformer that keeps position in line with animated topology.
It's cool because this way you can actually get some really nasty collisions out of the way in ways that would be really hard to sort out procedurally.

But you know what? I've got a variation of this.

	v@P2 = point(1,"P",0);
	float dist = distance(@P,v@P2);
	float ratio = chf("ratio");
	float min = chf("min");
	float max = chf("max");
	float velocity = point(2,"v",@ptnum);
	float speed = point(2,"speed",@ptnum);

	@P = @P + (@P - v@P2) * -ratio * fit(dist,min,max,1,0);
	@P = @P + normalize(velocity) * speed / 24;

Almost the same thing, except it takes consideration of the distance from a very specific point that you can set yourself!
This one was very much work in progress I made in an attempt to get very specific collisions out of the way but heh, doing it based on sdf or a point or whatever helps you achieve your dreams!

------------------------------------------------------

Join node Alternate

A long time ago I had quite a bit of a problem with a specific thing, essentially I couldn't really put together lines into a single polyline. Today I know what was going on and there are a few techniques of addressing this.
But at the time I decided to be a genius and create my own... hmm, what was it? Jesus, I'm such a genius that I prefer my own tools over legacy houdini, wololo.

Anyway, two different wrangles one after the other: Prim wrangle first:

	int primpoints_count[] = primpoints(0,@primnum);
	i@primpoints_count = len(primpoints_count);

point wrangle now

	int primpoints_count[];
	int sum_of_points[];
	int sum_of = 0;
	float offset;
	int last_points;
	int first_points;

	for(int i=0; i<@numprim; i++){
		int sum = prim(0,"primpoints_count",i);
		push(sum_of_points,sum);
		push(primpoints_count,sum);

		int sum_of_points_last = sum_of_points[-1:][0];
		sum_of += sum_of_points_last;

		addprim(0,"polyline",sum_of-1,sum_of);
	}

	//addprim(0,"polyline",0,sum_of-1); //if you want to close it up
	last_points = sum(primpoints_count[:-1]);
	first_points = sum(primpoints_count[1:]);

	float offset_ramp = chf("offset_ramp");
	offset = last_points - first_points * offset_ramp;
	i@offset = int(offset);

Then chuck down a polypath, voila doesn't matter whether you sort your stuff or if the uvw stuff is pointing the wrong way, no reverse nodes no nothing - this will sort you out mate.

Looking back, though, this really played a number on my head - took a fair bit to crack and hey, that's what always happens when you're not in the coding game non-stop. But it's cool to crack some heads with code every now and then, except this one was really tough.
I mean, array slicing gets weird at times. And I didn't really know you can actually use syntax like this, was so surprised I didn't get thrown errors when I trial and errored this.
Also pushing values into an array stored elsewhere from a for loop with slicing arrays is just something I wasn't really introduced per say back in higher education anyway.
My background isn't computer science but I can sort out problems like these every now and then ;)

------------------------------------------------------------

Hashmaps aka Dictionaries

There was a cool concept a good many years ago that I've learned as part of a Information Visualisation with Processing course which was hashmaps which is like arrays but the keys are strings.
This could be particularly useful for a company which doesn't exactly have a pipeline set to execute based on current shot and needs something to execute properly in switches in sops/lops/rops.

Detail Wrangle

	d@hashmap = set(
	"shotname_1",1,
	"shotname_2",2);

	s@shot_ref = chs("/obj/RefNode/ShotCode");
	string shot_ref = s@shotref;

	i@switcheroo = getcomp(d@hashmap,shot_ref,0);

You gonna ask me what does it do? Well, shiiiii, you change numbers and set defaults based on shot-context. What does it give you? Well, superpowers mate - you can essentially create your own dictionary of how you want the values to change based on shot and reference it!

--------------------------------------------------------------