+++
date = '2025-11-17T11:29:26+11:00'
draft = true
title = 'State machines in Zephyr using boost::sml'
author = 'Dean Sellers'
tags = [
    "C++",
    "Zephyr",
    "state-machine",
    "embedded"
]
+++

# State machines in Zephyr using C++ boost::sml

The [Zephyr OS](https://docs.zephyrproject.org/latest/introduction/index.html) does provides a [state machine framework](https://docs.zephyrproject.org/latest/services/smf/index.html) which is really flexible and easy to use, I don't really have anything against it. I did have the chance recently though to do some evaluations of state machine libraries for a C++ codebase and came to the conclusion that the [boost::smp](https://github.com/boost-ext/sml/tree/master) library was a great fit for what was needed at the time. If implementing a state machine pattern in a C++ codebase is on your radar I would encourage you to take the time to [review the rationale](https://boost-ext.github.io/sml/index.html#acknowledgements) and consider it.

I got curious about employing the same library in Zephyr based projects, and wrote a bit of code to test it out. Before you shout *embedded C++ BOOST, who are you kidding*, this is a header only, really resource efficient approach. Modern C++ brings a lot of advantages to embedded system programming and has been rather unfairly seen as *the bloated cousin of C* in some circles, particularly when you add the word *boost*. While this sentiment is probably less popular now, I still run into it from time to time. I thought a quick dive into using modern C++ with a modern (C based) embedded OS might be interesting.

I won't spend a lot, well any really, time going through why/what/how a state machine. There are plenty of resources around to help you answer these questions, and I am sure you'll find out eventually *it's all state machines, all the way down*.

## The model

The simple state machine I decided to model is a light switch, with a dimming function. There are four input events

1. `On` - Turn the light on, at maximum brightness
1. `Off`- Turn it off
1. `Up` - Increase the brightness, to a maximum, if off restore the last brightness
1. `Down`- Decrease the brightness, if at minimum turn it off

There are two states, `On` and `Off` and a brightness value. As a UML diagram it looks like this ...

![UML diagram of dimmable light state machine](/dimmed-light.png "UML diagram of the dimmable light state machine")

Full disclosure, I used the [PlantUML example from boost::sml](https://boost-ext.github.io/sml/examples.html#plant-uml-integration) to generate the PlantUML code from the state transition table .. self documenting! With a bit of a tidy up the PlantUML generated looked like this.

```plantuml
[*] -> off
off -> on : e_ON /bright=5
off -> on : e_UP /last_bright?bright=last_bright:bright=1
on -> on : e_UP [bright != 5]/bright+=1
on -> off : e_OFF /bright=0
on -> on : e_DN [bright > 1]/bright-=1
on -> off : e_DN [bright == 1]
off: entry bright=0
on: exit last_bright=bright
```

## Code

My github page has the code in two separate repositories if you want to try it out. The [first](https://github.com/d-o/sml) is a fork of the `boost::sml` repo where I have added a `/zephyr` directory with a few bits to allow the library to be imported as a [Zephyr module](https://docs.zephyrproject.org/latest/develop/modules.html#). You don't really have to do much with this, but you could have a look if you are interested in how to make a simple Z module.

The other one contains the Zephyr application, and west manifest to import Zephyr and the boost::sml library. It's [README](https://github.com/d-o/z-sml) gets you setup to build and test it out.

The meat of the job is done by the following, from `main.cpp`, 60 odd lines of code.

```cpp
// Four events
namespace {
	struct e_ON { static constexpr std::string_view name = "On"; };
	struct e_UP { static constexpr std::string_view name = "Up"; };
	struct e_DN { static constexpr std::string_view name = "Down"; };
	struct e_OFF { static constexpr std::string_view name = "Off"; };

// Data for each light context
struct lt_ctx {
	int bright = 100000;
	int last_bright = 0;
};

struct dimmable_machine {

    auto operator()() const noexcept {
	using namespace sml;

	// clang-format off
	return make_transition_table(
	 *"Off"_s + event<e_ON> / [](lt_ctx& lc){ lc.bright = 5; } = "On"_s
	, "Off"_s + event<e_UP> / [](lt_ctx& lc){ lc.bright = lc.last_bright?lc.last_bright:1;} = "On"_s
	, "On"_s + event<e_UP> [!![](lt_ctx& lc){return lc.bright < 5; }] / [](lt_ctx& lc){ lc.bright++; }
	, "On"_s + event<e_DN> [!![](lt_ctx& lc){return lc.bright > 1; }] / [](lt_ctx& lc){ lc.bright--; }
	, "On"_s + event<e_DN> [!![](lt_ctx& lc){return lc.bright == 1; }] / [] {} = "Off"_s
	, "On"_s + event<e_OFF> / [] {} = "Off"_s
	, "Off"_s + sml::on_entry<_> / [](lt_ctx& lc){ lc.bright = 0; }
	, "On"_s + sml::on_exit<_> / [](lt_ctx& lc){ lc.last_bright = lc.bright; }
	);
	// clang-format on
    }
};
}  // namespace

int main() {
	using namespace std;
	// Event instances
	e_ON ON;
	e_OFF OFF;
	e_UP UP;
	e_DN DN;

	// A vector of events to run
	vector< variant<e_ON, e_OFF, e_UP, e_DN> > evts = {
		ON, OFF, UP, /* 5, 0, 5 */
		DN, DN, DN, /* 4, 3, 2 */
		OFF, UP, UP, UP, UP, UP, /* 0, 2, 3, 4, 5, 5 */
		DN, DN, DN, DN, DN, DN, /* 4, 3, 2, 1, 0, 0 */
		UP, OFF, ON, OFF /* 1, 0, 5, 0*/
    	};

    	// Context data for a dimmable light
    	lt_ctx ct{};
    	//Create a light with the context data
    	sml::sm<dimmable_machine> sm{ct};

    	// Run through the states ...
    	sm.visit_current_states([](auto state) { cout << "Light started @ " << state.c_str() << std::endl; });
    	for(auto& v: evts) {
		visit([&sm](const auto& e){sm.process_event(e); cout << "Event \"" << e.name; }, v);
		cout << "\"\tLevel " << ct.bright << "\tState ";
		sm.visit_current_states([](auto state) { cout << state.c_str() << std::endl; });
    	}
}
```

## Output and analysis

Running the above in `qemu` gives the following output[^1];

```plaintext
*** Booting Zephyr OS build v3.7.0 ***
Light started @ Off
Event "On"	Level 5	State On
Event "Off"	Level 0	State Off
Event "Up"	Level 5	State On
Event "Down"	Level 4	State On
Event "Down"	Level 3	State On
Event "Down"	Level 2	State On
Event "Off"	Level 0	State Off
Event "Up"	Level 2	State On
Event "Up"	Level 3	State On
Event "Up"	Level 4	State On
Event "Up"	Level 5	State On
Event "Up"	Level 5	State On
Event "Down"	Level 4	State On
Event "Down"	Level 3	State On
Event "Down"	Level 2	State On
Event "Down"	Level 1	State On
Event "Down"	Level 0	State Off
Event "Down"	Level 0	State Off
Event "Up"	Level 1	State On
Event "Off"	Level 0	State Off
Event "On"	Level 5	State On
Event "Off"	Level 0	State Off
```

I am not going to spend a lot of time on the performance aspects, although it would be interesting to compare it to Zephyr's bundled SMF, and there is a good comparison of different approaches [on boost::sml's page](https://boost-ext.github.io/sml/benchmarks.html). In terms of resource use though the ram and rom reports are a bit interesting. They are too big to fully reproduce but if you have built the demo yourself you can run them using west, `west build -t ram_report` and similarly `west build -t rom_report`.

Excerpts from the rom report show not much impact of including `boost::sml` and the whole `main` function.

```
├── WORKSPACE                                      586   0.42%  -
│   ├── boost-sml                                  126   0.09%  -
│   │   └── include                                126   0.09%  -
│   │       └── boost                              126   0.09%  -
│   │           └── sml.hpp                        126   0.09%  -
│   │               ├── _ZN5boost..vRKSB_           38   0.03%  0x80003390 text
│   │               ├── _ZN5boost..vRKSB_           38   0.03%  0x800033b6 text
│   │               └── _ZN5boost..vRKSB_           50   0.04%  0x8000335e text
│   └── smp_z_demo                                 460   0.33%  -
│       └── src                                    460   0.33%  -
│           └── main.cpp                           460   0.33%  -
│               └── main                           460   0.33%  0x800033dc text
```

I don't pretend to fully understand the mechanics but using the power of C++ meta programming results in the whole machine the `WORKSPACE` total being 586 bytes in code. That seems impressive, of course there is overhead in introducing the C++ libs, but if your code was using it anyway, you have paid that price.

The ram report has nothing to say about any storage being used, which makes sense as in the code above I've got all of the instance variables on `main`'s stack. Moving the events and the context into global variables and running the ram report again shows;

```
│   ├── _ZGVZN5boost..3str                 8   0.04%  0x80021010 bss

│   ├── _ZZN5boost..3str                  42   0.19%  0x80021480 bss

├── WORKSPACE                             20   0.09%  -
│   └── smp_z_demo                        20   0.09%  -
│       └── src                           20   0.09%  -
│           └── main.cpp                  20   0.09%  -
│               ├── ct                     8   0.04%  0x80022be0 datas
│               └── evts                  12   0.05%  0x800214b8 bss
```

So ... maybe 70 odd **bytes** in RAM.

## Conclusions

This is just all fairly back of the envelope, but it doesn't seem that C++ has to be the hungry resource sucking beast that it gets made out to be. Of course this is a trivial example, but not that far removed from the sort of machines an embedded system has to run. If you can introduce a pattern like this for not much overhead, with the added benefits of clarity in code, ease of maintenance and strong type and memory safety ... seems like something we should all at least look at.


[^1]: Probably could have spent more time on formatting that, but I am an embedded guy, wallpaper has never been my thing.