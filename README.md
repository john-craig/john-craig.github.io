4/5
	Over the past weekend I have mostly been grappling with an little tool I use for synchronizing all of my to-dos. There are two places I track to-dos; one is a Joplin notebook, and the other is a LiberOffice document.
	Typically when I complete a task, I want it removed from Joplin, and recorded permanently in the document, so that I can go back and review it in the future. For the most part I both create new tasks and mark them completed in Joplin, because I have the application for it on my phone.
	What I have done so far is build out a Python program that can interact with both Joplin and LibreOffice through some libraries and extract the information I want into a standard format.
	In the case of Joplin this involves looking through a specific notebook and pulling out all the entries which are to-dos and have tags set on them. I use tags to specify the day that the item is supposed to correspond to-- there is a feature for setting due dates on to-dos in Joplin, but that causes an alarm to go off on my phone, which I sort of hate.
	For LibreOffice this is a little more complicated. I have the documents I use for tracking my tasks arranged in a certain format, like so:

```
[Weekday]
	Tasks:
    	• [Incomplete To-Do]
	Deeds:
    	• [Complete To-Do]
```

This requires more customized parsing, although it’s a bit easier to determine the day of the week, for example.
	At the moment the program builds a Python dictionary with fields for the title, date, and completion status of each item. This abstraction obviously makes them much easier to compare and otherwise work with. The step I am on now is applying modifications derived from these comparisons back to the notebook and document from which they were parsed.

4/6
	Well, I was going to spend the day continuing to detail all the little intricacies of my Agenda Manager, but I had a minor catastrophe. I was looking to get a bunch of Github repositories made and start uploading all the local copies of my repositories, just in case of, you know, a hard drive brick or something like that.
	While I was at it I was also going to create a dotfile repository. Because the hidden files in your user directory typically contain all the configuration information for the software that you use, creating a github repository that contains those files is a good way of “backing up” your configurations.
	Unfortunately, creating a dotfile repository properly can require a somewhat artistic usage of your gitignore, and to be honest gitignores are not something I’ve always been great with myself. Well, in tinkering around with my gitignore and seeing what files would and would not get added when I did a git add, I sort of, kind, maybe, recursively deleted all the dotfiles in my home directory.
	So actually the exact reason you might want a dotfile repository-- that happened.
	After it became clear that there was no way to make git regurgitate my precious configurations, I chose to look at this as an opportunity to start fresh and take a slightly more cohesive approach to my desktop. My theme choices had gotten to be fairly… eclectic, to put it gently. Now I can at least pick one palette and try to stick with it more strictly than I have been.
	There were a few critical repairs to be made. My whole desktop won’t start without a properly-configured .xinitrc, for instance, and some of my aliases were pretty complex. There are a few that would actually run utility variations of the agenda manager to find the right path to the current week’s log, for instance.

4/7
	Spent a little while repair all the fallout from that mistake. Still not totally done, but what I did take some time to do was write a script to automate the git commit-and-push process. A saner person might have learned their lesson after being bit by git once and stayed far away from its cave, but I still have designs to get a collar on the thing.
	The thing I spent the longest time grappling with wasn’t really git’s functionality itself, but rather authenticating the pushing process. I don’t really have a credential manager set-up on my desktop right now. I keep putting it off until I can get a self-hosted Bitward instance running, and, well, maybe I should just find a short-term solution for it in the meantime, because that’s not happening anytime soon.
	But fun fact, no.
	So to deal with authentication I need to provide the credentials in the git-sync script myself. Now, I have been considering making some kind of utility to extract usernames and passwords from my KeePass database-- the current software I use for password management-- but that was sort of out-of-scope for what was supposed to be a fairly simple automation script, so…
	Well, basically I got it working inelegantly. I can always go back and make improvements later, right?

4/8
	Spent a bit of time tinkering around with my homelab. It currently encompasses the might extent of a singular RaspberriPi board, although I have grand designs, as usual.
	There are a few self-hosted services that I want to have up and running on the Pi, and at the moment the highest-priority is NextCloud.
	Right now I have NextCloud up and running on my desktop, which is useful because I can synchronize files from my phone and laptop, but… if my desktop were to brick, or if I were to do something stupid like I did earlier this week and delete large swathes of the filesystem by accident, I would lose everything.
	Before I can configure the actual services I want to run, though, I need to properly configure the self-hosted stack. This consists of a proxy service, specifically Nginx Proxy Manager, and Portainer, which will handle the services, both running in Docker Compose. Now, I know a decent amount about Docker, less about Docker Compose, and less still about Nginx, so this is a bit of a learning opportunity.
	The set-up I am going for would be to have a domain name with SSL certificates that I can connect to on my local network, and though which I can access my various services. Based off my current knowledge, this set-up is not going to be happening any time soon, but that’s why I work on these side-projects: to learn how to stuff that I don’t know super well.

4/9
	But, sometimes I have to spend time on required projects, i.e. schoolwork. At the moment I am in a class for mobile development, which is obviously a pretty valuable skill to learn. I would prefer a little more freedom about the project I am working on in said class, but I suppose it is better to stick to the curriculum and pursue my own ideas on my own time.
	So, it’s a simple game, I think there’s even a classification for it, a “three-lan shooter”, where the enemies approach you from one of three directions, and you have to aim your shooter (in this case a crossbow) in a given direction to fire projectiles and hit them.
	The next stage of the project will be to integrate phone sensors in this application; at least two of them. I will probably use gyroscopic input to all the user to aim their crossbow by tilting their phone, but for the other sensor integration I’m less certain. I kind of want to make it something really weird, like a temperature sensor that makes the enemies move faster the hotter it is physically. Or I could just do something easy like having the crossbow fire when the microphone detects loud audio input.
