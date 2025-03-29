---
tags:
  - iku/check
  - iku/draft
---
# Formatting Decisions
This page is about my decisions on how to format the document so it's easier for me to read as I design my game.


## Concepts and Code
- I went with *Italics* for concepts.
	- Try to write the singular version and plural versions in the same page / paragraph so using find commands work a bit better.
- Try saying "with code" or "in code" when using `LiteralCode` formatting so the reader remembers to shift their frame of mind to editing code in their IDE.


## While Writing
### Using shorthand
- For abbreviations, (like GE) I did not format, as you might encounter them elsewhere and I felt like it would be an escape from too many *concepts* overloading the reader.

### Giving Casual Commentary
> I kept comments for my casual commentary.


### Checking off things to look into
- [ ] Usually I need to check the code, and want to do all that last / at once.


### Keeping sub-headers similar sounding
Notice how the H3's (*Header 3*) all have "-ing" used?


## Tagging
WIP. Used to get around in *Graph View*.


Most pages will have: 

#GAS-System/Built-In/Effect 
or
#GAS-System/Built-In/Concept


Most Pages will also have the following to show parenting:

#GAS-System/Effect 
#GAS-System/Ability
#GAS-System/Attribute
#GAS-System/AttributeSet




### Example

If your random deviation is small, most players won't notice that the sequence is the same every game and using the activation prediction key as the *random seed* should work for you. If you're doing something more complex that needs to be hacker proof, perhaps using a *Server Initiated* *GameplayAbility* would work better where the server can create the prediction key or generate the *random seed* to send via an event payload.


vs



If your random deviation is small, most players won't notice that the sequence is the same every game and using the activation prediction key as the *random seed* should work for you. If you're doing something more complex that needs to be hacker proof, perhaps having the *Server* initiate the *Gameplay Ability* would work better where the server can create the prediction key or generate the *random seed* to send via an event payload.



