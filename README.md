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

	f[]@length;

	for(int i=0;i<@numpt-1;i++){
	        vector pt1 = point(0,"P",i);
	        vector pt2 = point(0,"P",i+1);
	        float measure = distance(pt1,pt2);
	        push(@length,measure);
	}

	f@radius = (sum(@length))/(2*$PI);

It's important that this code affects only a ring, plenty of ways of extracting just one ring (although always troublesome whenever dealing with an entire series of cables etc.), otherwise you'll kind of end up calculating the length of all edges in your geometry. I mean, you could do that but why not just use a measure sop instead...

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

	for(int i=0; i<@numprim; i++){
		int sum = prim(0,"primpoints_count",i);
		push(sum_of_points,sum);
		push(primpoints_count,sum);

		int sum_of_points_last = sum_of_points[-1:][0];
		sum_of += sum_of_points_last;

		addprim(0,"polyline",sum_of-1,sum_of);
	}

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

Procedural Attribute Adjustment with Hscript (and maths)

Kind of taking the lead from my $FSTART and $FEND idea I'd also introduce some hscript instead of keyframing. Keyframing I always found to be a bit of a pain in the butt, especially in production since you'd end up really refining those keys until the asset is final and it's so finecky. If you want a very specific motion, sure mate but you can also do some cool ramps in vex to procedurally adjust some cool motion. But if you need something quick and dirty in the attrib field? Well.

	fit(@Frame,$FSTART,$FEND,0,1)

Yup, that's all you need my brethren. You can feed this directly into possibly any parameter and actually suit it to your needs and introduce some other procedural sauce. Maybe some sin waves or some other cool motion? Be my guest. I'd also do cool stuff like this.

	fit(@Frame,$FSTART,$FEND-120,0,1)+fit(@Frame,$FSTART+120,$FEND,0,-1)

Again, straightforward but you can do all sort of cool procedural motion with this and you can look into the channel to see exactly what you're up to! ...or use a tool like wolframalpha to check some crazier mathematical equations. I obviously didn't do this all the time but at some stage when doing CFX, I noticed that ditching keyframes is the way forward (especially if working across many shots!).

-------------------------------------------------------------------

Culling

Big part of your job at the end of day is, well, pushing stuff through efficiently. There's a million ways to achieve the final effect, probably the ones that take less time and make your worflow optimized will trump everything else, while keeping the flexibility and looks right obviously. Sometimes to reach the right feel, you need the numbers and the numbers need to be pushed radically upwards but there's only so much that can be stuffed in your RAM, especially during simulation. So what should you do? Cull the hell out of it!

There's a few techniques I've seen before, Mr. Knipping suggested in regards to point-based stuff to apply a uv texture and blast this way. I experimented a bit with a volume frustum, which I think I should explore at a few angles just in case (although it's given me plenty of headaches) but at the end of day, culling by toNDC is best.

	string camera = chsop("camera");
	vector campos = toNDC(camera,@P-chv("offset"));

	if(camera.x > 1.05 || camera.x < -0.05 || camera.y > 1.05 || camera.y < -0.05){
		@group_blah = 1;
	}

Then blast it. Don't use it in a solver as a pop/geo wrangle, as it does some nasty things, so -> sopsolver, gas intermittent to execute it once every frame and voila, should be good.
Interestingly, you can do something very similar without much overhead from what I noticed for volumes, even in a pyrosolver when chucked into a gaswrangle, it's weird but hey, I've done some very beautiful culling this way. Also, steal a vdbcombine from one of the vdb sops that optimizes vel. If you don't wanna, you can use a vdbcombine, plug same dop geo into both inputs, groupA: vel, groupB: density, make an activity intersection and you know what? Your pyro vel will go as far as density goes until it's blasted. In theory, it might cause some issues but I haven't seen any in my use cases and likely in most cases you won't either but this will keep your sims nicer, cuter, smaller etc. etc.

-------------------------------------------------------------------

Calculate bounding box from packed objects

This is a bit of a smaller thing but I didn't know volumes of packed objects don't get calculated by default and in certain circumstances, this can come in quite rather handy actually. I couldn't get measure sop to work and other sop-based methods failed weirdly, so... instead!
Do a primwrangle and:

	float bounds[] = priminstrinsic(0,"bounds",0);
	f@vol = abs(bounds[0]-bounds[1])*abs(bounds[2]-bounds[3])*abs(bounds[4]-bounds[5]);

This will calculate a rough volume of your packed objects. Is this ideal? No, as it only approximates your thing and if you'd like precise measures, I guess a measure sop on unpacked objects will be your friend here BUT!
When you're working with say, thousands, tens of thousands, millions of packed geometry, you might want to have a quick and dirty way of purging smaller stuff, so you can create logic ya know.
For instance, anything smaller than, say, 1 quadratic meter, please leave my scene etc. Anything larger far away, that too and afterwards the rest? Please make it an sdf hehe.

-------------------------------------------------------------------

Wrangle remap to min/max values

Ever had a problem, you have some values but don't have min/max values so, making logic is being difficult? I know you can use a node to calculate min/max values once but you might want to do it dynamically instead.
Detail wrangle here I come:

	float value[];

	for(int i=0; i<@numpt-1; i++){
		float pt_value = point(0,"some_value",i);
		float val = pt_value;
		append(value,val);
	}

	f@value_min = min(value);
	f@value_max = max(value); //if you want to do stuff to it, give it a clamp to 0,1 hehe
	//f[]@value = value; //if you want this exposed, which you might want

Now you can do any sort of magic you'd like based on the min/max values, pronto

-------------------------------------------------------------------

Heightfield wrangle (mix masks/height conditionals)

This one's pretty straightforward but I found myself needing to use heightfields to do some work. Heightfields are amazing when used as colliders, Houdini does not quite use them as part of their usual solver coolness and rather prefers to rely on VDB-based plug-ins or surface collisions. First pretty handy, second bit less ideal for performance reasons as I found from RnD on Furiosa. My sup recommended giving a whirl with heightfields, which requires hacking the solver and feeding in terrain object and hey, it's pretty sweet.

Anyway, as you use heightfields, you kind of need to stick to them and the amount of nodes available can be a bit limiting, even when messing around with masks. The simple idea is to use a wrangle to do just what you want but these are heightfields, so how exactly do I...? Well, it's rather simple after all but not so obvious if you think about it. Here's a sampling thing and some basic filtering example.

	@mask -= volumesample(1,"mask",@P);
	if(@mask<0.001){
	@mask += volumesample(2,"mask",@P);
	}
	if(@height<50 || @height>25){
	@mask = 0;
	}

-------------------------------------------------------------------

Volume operations based on a variable

This one also not so obvious. Doing bespoke operations on volumes is slightly tricky, as they don't seem to be as easily viewable, in terms of what operations you're really feeding them. Usually I'd just turn them into points and then see what a pointwrangle does and check geo spreadsheet to know what's going on. I had a task, in which a volumes in the shot had to be culled by a falloff and also in certain areas, rather easy task to be fair but how does one do this? Well, it's like in a pointwrangle really, stuff seems to work on volumes too which is cool.

I did this thing a few times now and I dislike re-inventing the wheel, I always get my mind blown by the distance from nearest point, kinda looks ok but once you start looking at it, weird distances appear and I know why - because I'm not pointing to the right near points from the right context! It happens folks.

	float dist = distance(@P,point(1,"P",0));
	float dist_remap = fit(dist,1000,0,1,0);
	float ramped = chramp("remap_this",dist_remap);
	
	@density *= ramped;
	
	int near = nearpoint(2,@P,1000);
	float dist2 = distance(@P,point(2,"P",near));
	float dist_remap_2 = fit(dist2,1000,0,1,0);
	float ramped2 = chramp("remap_ya",dist_remap_2);
	
	@density = @ramped2;

-------------------------------------------------------------------

Aggregate strings, find them afterwards

Ever had some gnarly and complicated scene with tons of stuff flying everywhere and the need to clean this up, perhaps bit more procedurally? Cause like, ain't nobody keen to doing 3D roto, right? Well, drop down a detail wrangle and with a timeshift at, probably the end of the scene, drop down this:

	s[]@names;
	
	for(int i=0;i<@numpt-1;i++){
	string nam = point(0,"name",i);
	append(@names,nam);
	}

You'll pack all the existing names into a detail attribute. What can you do afterwards with this? Well, something like this:

	s[]@names = detail(1,"names");
	
	if(find(@names,@name)<0){
	}
	else{
	removepoint(0,@ptnum,1);
	}

What will this do? Oh, it will just try and find whether the name exists or not and if it doesn't, boom. It can be really helpful if you need a consistent point count or RBD objects with consistent naming convention, at the very least to make sure moblur doesn't go crazy or stuff doesn't appear out of nowhere. Weird things can happen and a wrangle like this can prevent this from happening.


-------------------------------------------------------------------

Misc

-------------------------------------------------------------------

Playing back .exrs with MPV

Here's a little trick. Since I switched to Windows 11 on my laptop, I lost my ability to view .exrs with irfanview - which wasn't an ideal solution. I could use Nuke, Adobe Premiere (cumbersome), maybe Xstudio, there are some alternatives out there or even new Houdini Cops. But, there is one more solution that I've found pretty neat and that's using MPV, which I use to playback any videos, it's customization options are just incredible. And you can you use it to playback things from a command line.

So, in cmd point to where you MPV lives like

	cd Desktop/mpv

 And then you can do some cool magic like, playback an .exr sequence.

 	mpv "mf://C:\Users\VeryCoolUser\Desktop\Project\render\AmazingRender.%04d.exr" --demuxer-lavf-format=image2

It's not magical, the %04d is the frame number in xxxx format.
I'm sure anyone after computer science with foo through issues like this in no-time.
But I find it pretty handy to just playback anything I'd like from a command line - and there's heaps of options on their website. Check it out!

Heyyy, what you want more? Be patient mate, more will arrive in due time as I figure stuff out ^^

-----------------------------------------------------------------------

Audio to spectrum conversion

Using ffmpeg, you can convert your music into something that you can read in houdini without using chops. This will give a 24fps output, which uses some extra post-processing to make sure the images align with the music. Do this with ffmpeg, make sure you have folders created in the ffmpeg bin directory.

	ffmpeg -i "C:\Users\YoPath\Music\YoSong.mp3" -filter_complex "[0:a]channelsplit=channel_layout=stereo[L][R];[L]showspectrum=s=1x1024:scale=lin:start=0:stop=20000:slide=replace:color=intensity,format=gray16be,fps=24[left];[R]showspectrum=s=1x1024:scale=lin:start=0:stop=20000:slide=replace:color=intensity,format=gray16be,fps=24[right]" -map "[left]"  -fps_mode passthrough left\frame_%06d.png -map "[right]" -fps_mode passthrough right\frame_%06d.png

The result will be 1x1024 greyscale images that you can read back in houdini in lines
