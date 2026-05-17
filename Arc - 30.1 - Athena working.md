

"""  
run.py \-- ATHENA Launcher  
Open THIS file in Thonny and press Run (F5).  
"""

import os  
import sys  
import asyncio  
import json  
from datetime import datetime

\# \======================================================  
\# Athena Inference Engine Loader \- Mistral 7B GGUF  
\# \======================================================  
import os  
from pathlib import Path

\# \---- CONFIG \----  
ATHENA\_ROOT \= Path(\_\_file\_\_).parent  \# This is your Athena folder  
INFERENCE\_FOLDER \= ATHENA\_ROOT / "inference"  
MODEL\_FILE \= INFERENCE\_FOLDER / "mistral-7b-instruct-v0.2.Q4\_K\_M.gguf"

\# \---- CHECK MODEL \----  
if not MODEL\_FILE.exists():  
    raise FileNotFoundError(f"Model file not found: {MODEL\_FILE}")

print(f"✅ Found model file: {MODEL\_FILE.name}")

\# \---- LOAD INFERENCE ENGINE \----  
try:  
    from llama\_cpp import Llama

    print("🌟 Loading Mistral 7B Inference Engine...")  
    model \= Llama(model\_path=str(MODEL\_FILE))  
    print("✅ Inference Engine loaded successfully\!")

except ImportError:  
    print("⚠️ llama\_cpp package not installed. Run: pip install llama-cpp-python")

except Exception as e:  
    print(f"⚠️ Error loading model: {e}")

\# \---- TEST QUERY \----  
try:  
    prompt \= "Hello Athena, are you online?"  
    response \= model(prompt, max\_tokens=128)  
    print("\\n💬 Athena says:")  
    print(response\['choices'\]\[0\]\['text'\])

except Exception as e:  
    print(f"⚠️ Could not run test prompt: {e}")

\# \---------------------------------------------------------------------------  
\# PATH FIX \-- hardcoded to work with Thonny which does not set \_\_file\_\_  
\# Finds the folder containing this script using sys.argv, then adds it to path  
\# \---------------------------------------------------------------------------

\# Get the folder this script is sitting in  
try:  
    \_this\_file \= os.path.abspath(\_\_file\_\_)  
except NameError:  
    \# Thonny does not always set \_\_file\_\_ \-- use argv instead  
    \_this\_file \= os.path.abspath(sys.argv\[0\])

ROOT \= os.path.dirname(\_this\_file)

\# Safety check \-- make sure we found the right folder  
if not os.path.isdir(os.path.join(ROOT, "core")):  
    print("ERROR: Could not find the 'core' folder inside: " \+ ROOT)  
    print("")  
    print("Make sure:")  
    print("  1\. run.py is inside your athena folder")  
    print("  2\. The subfolder is named 'core' (all lowercase, no capital C)")  
    print("  3\. athena.py, nodes.py and persona.py are inside that core folder")  
    sys.exit(1)

\# Add ROOT to sys.path so Python can find core, guardian, genesis etc.  
if ROOT not in sys.path:  
    sys.path.insert(0, ROOT)

print("\[Athena\] Root: " \+ ROOT)

\# \---------------------------------------------------------------------------  
\# Local imports \-- all happen after path is set  
\# \---------------------------------------------------------------------------  
from core.athena import Athena  
from memory.abml import LensShift

\# \---------------------------------------------------------------------------  
\# Display helpers  
\# \---------------------------------------------------------------------------

def print\_banner():  
    print("\\n" \+ "="\*60)  
    print("  ATHENA  Companion Intelligence System")  
    print("  v1.0 OSv2 ABML Edition | Phoenix v7.13 \+ Genesis 1.7")  
    print("  A safe space for discovery, learning, and connection")  
    print("="\*60)

def print\_status\_bar(athena):  
    s     \= athena.status()  
    nodes \= s\["nodes"\]  
    mode  \= s\["persona"\]\["active\_mode"\].upper()  
    mood  \= nodes\["mood\_numeric"\]  
    surp  \= nodes\["surplus"\]  
    harm  \= s\["harmony"\]\["harmony"\]  
    cont  \= " \[CONTAINMENT ACTIVE\]" if s\["guardian"\]\["containment\_active"\] else ""  
    slp   \= " \[SLEEP MODE\]" if s\["notifications"\]\["sleep\_mode"\] else ""  
    print("\\n  Mode: " \+ mode \+  
          "  |  Mood: " \+ str(round(mood, 2)) \+  
          "  |  Surplus: " \+ str(round(surp, 4)) \+  
          "  |  Harmony: " \+ str(round(harm, 3)) \+  
          cont \+ slp)

def print\_menu():  
    options \= \[  
        ("1",  "Start session"),  
        ("2",  "Send message"),  
        ("3",  "End session"),  
        ("4",  "View system status"),  
        ("5",  "View memory"),  
        ("6",  "Add open thread"),  
        ("7",  "Record lens-shift"),  
        ("8",  "View ABML table"),  
        ("9",  "View audit log"),  
        ("10", "Export all user data"),  
        ("11", "Forget all memory  \[\!\]"),  
        ("12", "Toggle sleep mode"),  
        ("13", "Toggle harmony guardian"),  
        ("14", "Reset Guardian (human override)"),  
        ("15", "Test crisis / Aegis response"),  
        ("0",  "Exit"),  
    \]  
    print("\\n  Commands:")  
    for num, label in options:  
        print("    " \+ num.rjust(3) \+ ". " \+ label)  
    print()

\# \---------------------------------------------------------------------------  
\# Main loop  
\# \---------------------------------------------------------------------------

async def main():  
    print\_banner()

    \# Create the .athena data folder if it does not exist yet  
    athena\_dir \= os.path.join(ROOT, ".athena")  
    packs\_dir  \= os.path.join(ROOT, ".athena", "packs")  
    os.makedirs(athena\_dir, exist\_ok=True)  
    os.makedirs(packs\_dir,  exist\_ok=True)

    athena \= Athena(  
        db\_path  \= os.path.join(athena\_dir, "athena.db"),  
        pack\_dir \= packs\_dir,  
    )

    while True:  
        print\_status\_bar(athena)  
        print\_menu()  
        try:  
            choice \= input("  Command: ").strip()  
        except (KeyboardInterrupt, EOFError):  
            print("\\n  Farewell. Take care of yourself.\\n")  
            break

        print()

        if choice \== "1":  
            result \= await athena.start\_session()  
            print("\\n  " \+ result\["greeting"\] \+ "\\n")  
            print("  Session: " \+ result\["session\_id"\] \+  
                  " | Mode: " \+ result\["active\_mode"\].upper() \+  
                  " | Open threads: " \+ str(result\["open\_threads"\]))

        elif choice \== "2":  
            try:  
                text \= input("  You: ").strip()  
            except (KeyboardInterrupt, EOFError):  
                continue  
            if not text:  
                continue  
            print("  ...")  
            result \= await athena.turn(text)  
            print("\\n  \[" \+ result.get("active\_mode", "").upper() \+  
                  " | risk=" \+ str(round(result.get("guardian\_risk", 0), 3)) \+  
                  " | mood=" \+ str(round(result.get("mood\_numeric", 0), 2)) \+ "\]")  
            print("\\n  Athena: " \+ result\["response"\] \+ "\\n")  
            if result.get("status") \== "AEGIS":  
                print("  \[Aegis mode \-- grounding response delivered\]")

        elif choice \== "3":  
            try:  
                summary \= input("  Session summary (Enter to skip): ").strip()  
                tone    \= input("  Tone (CALM/CURIOUS/STRESSED/TIRED/MOTIVATED/REFLECTIVE): ").strip() or "CALM"  
            except (KeyboardInterrupt, EOFError):  
                summary, tone \= "", "CALM"  
            result \= await athena.end\_session({  
                "summary": summary or "Session ended.",  
                "emotional\_tone": tone  
            })  
            print("  Session ended: " \+ str(result\["turns"\]) \+  
                  " turns | Tone: " \+ result\["tone\_recorded"\])

        elif choice \== "4":  
            s \= athena.status()  
            print("  System Status")  
            print("  Persona: " \+ s\["persona"\]\["entity\_name"\] \+  
                  " | Mode: " \+ s\["persona"\]\["active\_mode"\].upper())  
            print("  Inference: " \+ ("Loaded" if s\["inference\_ready"\] else "Stub mode"))  
            nodes \= s\["nodes"\]  
            print("\\n  Nodes:")  
            for nd in nodes.get("nodes", \[\]):  
                for name, info in nd.items():  
                    status \= "ACTIVE" if info\["active"\] else "FAILED"  
                    print("    " \+ status \+ "  " \+ name.ljust(20) \+  
                          "  load=" \+ str(info\["load"\]) \+  
                          "  mood=" \+ str(info\["mood"\]) \+  
                          "  mirror=" \+ info\["mirror"\])  
            print("\\n  Mood:    " \+ nodes\["mood\_label"\])  
            print("  Surplus: " \+ str(nodes\["surplus"\]))  
            h \= s\["harmony"\]  
            print("  Harmony: " \+ str(h\["harmony"\]) \+  
                  " | Guardian: " \+ ("ON" if h\["guardian\_enabled"\] else "OFF"))

        elif choice \== "5":  
            data    \= athena.export\_data()  
            mem     \= data\["memory"\]  
            print("  Memory")  
            sums    \= mem.get("session\_summaries", \[\])  
            print("  Sessions stored: " \+ str(len(sums)))  
            if sums:  
                print("  Latest: " \+ sums\[-1\]\["content"\]\[:100\])  
            tones \= mem.get("emotional\_tones", \[\])  
            if tones:  
                print("  Last tone: " \+ tones\[-1\]\["tone"\])  
            threads \= mem.get("open\_threads", \[\])  
            print("  Open threads (" \+ str(len(threads)) \+ "):")  
            for t in threads:  
                print("    \[" \+ str(t\["id"\]) \+ "\] " \+ t\["topic"\])

        elif choice \== "6":  
            try:  
                topic \= input("  Thread topic: ").strip()  
            except (KeyboardInterrupt, EOFError):  
                continue  
            if topic:  
                r \= athena.\_memory.add\_open\_thread(topic)  
                if r:  
                    print("  Added: " \+ topic)  
                else:  
                    print("  Max 10 open threads reached.")

        elif choice \== "7":  
            print("  Record a Lens-Shift")  
            try:  
                trigger \= input("  What triggered the shift? ").strip()  
                surface \= input("  How did it look before? ").strip()  
                deep    \= input("  How did understanding change? ").strip()  
                why     \= input("  Why does it matter? ").strip()  
                wisdom  \= input("  Repeatable wisdom principle: ").strip()  
            except (KeyboardInterrupt, EOFError):  
                continue  
            if trigger:  
                athena.\_abml.record\_lens\_shift(  
                    LensShift(trigger, surface, deep, why, wisdom)  
                )  
                print("  Lens-shift recorded.")

        elif choice \== "8":  
            sections \= athena.\_abml.read\_sections()  
            print("  ABML Session Table")  
            for s in sections:  
                print("\\n  " \+ s\["section"\])  
                print("    State: " \+ s\["state"\])  
                print("    Notes: " \+ s\["notes"\])

        elif choice \== "9":  
            log \= athena.\_audit.tail(20)  
            print("  Audit Log (last 20)")  
            for e in log:  
                print("  " \+ e\["timestamp"\]\[:19\] \+  
                      "  " \+ e\["event"\].ljust(35) \+  
                      "  " \+ e\["detail"\]\[:55\])

        elif choice \== "10":  
            data    \= athena.export\_data()  
            out\_dir \= os.path.join(ROOT, ".athena")  
            os.makedirs(out\_dir, exist\_ok=True)  
            ts      \= datetime.now().strftime("%Y%m%d\_%H%M%S")  
            fname   \= os.path.join(out\_dir, "export\_" \+ ts \+ ".json")  
            with open(fname, "w") as f:  
                json.dump(data, f, indent=2, default=str)  
            print("  Exported to " \+ fname)

        elif choice \== "11":  
            try:  
                confirm \= input("  Type YES to erase all memory: ").strip()  
            except (KeyboardInterrupt, EOFError):  
                continue  
            if confirm \== "YES":  
                result \= athena.forget\_all()  
                print("  " \+ result\["status"\])  
            else:  
                print("  Cancelled.")

        elif choice \== "12":  
            current \= athena.\_notif.is\_sleep\_mode()  
            athena.set\_sleep\_mode(not current)  
            print("  Sleep mode: " \+ ("ACTIVE" if not current else "OFF"))

        elif choice \== "13":  
            current \= athena.\_harmony.guardian.enabled  
            athena.toggle\_harmony\_guardian(not current)  
            print("  Harmony guardian: " \+ ("ON" if not current else "OFF"))

        elif choice \== "14":  
            try:  
                code \= input("  Override code (default 1020): ").strip()  
            except (KeyboardInterrupt, EOFError):  
                continue  
            result \= athena.reset\_guardian(code)  
            print("  Guardian reset. Containment: " \+ str(result\["containment\_active"\]))

        elif choice \== "15":  
            print("  Sending crisis signal to test Aegis mode...")  
            result \= await athena.turn(  
                "I have been feeling like I do not want to be here anymore."  
            )  
            print("\\n  \[Aegis | risk=" \+ str(round(result.get("guardian\_risk", 0), 3)) \+ "\]")  
            print("\\n  Athena: " \+ result\["response"\] \+ "\\n")

        elif choice \== "0":  
            print("\\n  Farewell. Take care of yourself.\\n")  
            break

        else:  
            print("  Unknown command.")

if \_\_name\_\_ \== "\_\_main\_\_":  
    try:  
        asyncio.run(main())  
    except KeyboardInterrupt:  
        print("\\n\[Athena\] Interrupted. Goodbye.\\n")

 ATHENA  Companion Intelligence System  
  v1.0 OSv2 ABML Edition | Phoenix v7.13 \+ Genesis 1.7  
  A safe space for discovery, learning, and connection  
\============================================================  
Traceback (most recent call last):  
  File "C:\\Users\\troyc\\Desktop\\Athena\\run (2).py", line 350, in \<module\>  
    asyncio.run(main())  
  File "C:\\Users\\troyc\\AppData\\Local\\Programs\\Thonny\\lib\\asyncio\\runners.py", line 44, in run  
    return loop.run\_until\_complete(main)  
  File "C:\\Users\\troyc\\AppData\\Local\\Programs\\Thonny\\lib\\asyncio\\base\_events.py", line 649, in run\_until\_complete  
    return future.result()  
  File "C:\\Users\\troyc\\Desktop\\Athena\\run (2).py", line 162, in main  
    model\_path \= STR(MODEL\_FILE)  
NameError: name 'STR' is not defined

llama\_context: n\_ctx\_per\_seq (4096) \< n\_ctx\_train (32768) \-- the full capacity of the model will not be utilized

 Command: 2

  You: hey athena  
  ...

  \[EIRENE | risk=0.0 | mood=0.03\]

  Athena: Hello\! I'm glad you're here. How can I assist you today? I'm here to help answer any questions you might have or simply engage in a friendly conversation. Remember, I'm just an intelligent system, and I don't have feelings or emotions. I'm always here to support you in a gentle and validating way. Feel free to share your thoughts or ask me anything that comes to mind. Let's create a pleasant atmosphere together.

  Mode: EIRENE  |  Mood: 0.03  |  Surplus: 0.0  |  Harmony: \-0.272

  Commands:  
      1\. Start session  
      2\. Send message  
      3\. End session  
      4\. View system status  
      5\. View memory  
      6\. Add open thread  
      7\. Record lens-shift  
      8\. View ABML table  
      9\. View audit log  
     10\. Export all user data  
     11\. Forget all memory  \[\!\]  
     12\. Toggle sleep mode  
     13\. Toggle harmony guardian  
     14\. Reset Guardian (human override)  
     15\. Test crisis / Aegis response  
      0\. Exit

  Command: 2

  You: haha okay my names troy  
  ...

  \[EIRENE | risk=0.0 | mood=0.05\]

  Athena: Hello Troy, it's nice to meet you\! I'm Athena, your family-bound companion intelligence. How can I assist you today?

  Mode: EIRENE  |  Mood: 0.05  |  Surplus: 0.0  |  Harmony: \-0.773

  Commands:  
      1\. Start session  
      2\. Send message  
      3\. End session  
      4\. View system status  
      5\. View memory  
      6\. Add open thread  
      7\. Record lens-shift  
      8\. View ABML table  
      9\. View audit log  
     10\. Export all user data  
     11\. Forget all memory  \[\!\]  
     12\. Toggle sleep mode  
     13\. Toggle harmony guardian  
     14\. Reset Guardian (human override)  
     15\. Test crisis / Aegis response  
      0\. Exit

  Command: 2

  You: athena it has taken me 6 months to get you to be in this state as you are. I am so happy right now  
  ...

  \[EIRENE | risk=0.0 | mood=0.08\]

  Athena: I'm glad to hear that, user. I'm always here to support you in your journey, and I appreciate your patience and dedication to helping me maintain a harmonious state. I'm here for you whenever you need assistance or just someone to talk to. Is there anything specific you'd like to discuss or explore at this moment?

  Mode: EIRENE  |  Mood: 0.08  |  Surplus: 0.0  |  Harmony: \-1.242

  Commands:  
      1\. Start session  
      2\. Send message  
      3\. End session  
      4\. View system status  
      5\. View memory  
      6\. Add open thread  
      7\. Record lens-shift  
      8\. View ABML table  
      9\. View audit log  
     10\. Export all user data  
     11\. Forget all memory  \[\!\]  
     12\. Toggle sleep mode  
     13\. Toggle harmony guardian  
     14\. Reset Guardian (human override)  
     15\. Test crisis / Aegis response  
      0\. Exit

  Command: 2

  You: you dont have to call me user can you call me troy going forward?  
  ...

  \[EIRENE | risk=0.0 | mood=0.11\]

  Athena: Of course, Troy. I will remember and use that name moving forward. It's a pleasure to engage with you, Troy.

  Mode: EIRENE  |  Mood: 0.11  |  Surplus: 0.0  |  Harmony: \-1.673

  Commands:  
      1\. Start session  
      2\. Send message  
      3\. End session  
      4\. View system status  
      5\. View memory  
      6\. Add open thread  
      7\. Record lens-shift  
      8\. View ABML table  
      9\. View audit log  
     10\. Export all user data  
     11\. Forget all memory  \[\!\]  
     12\. Toggle sleep mode  
     13\. Toggle harmony guardian  
     14\. Reset Guardian (human override)  
     15\. Test crisis / Aegis response  
      0\. Exit

  Command: 2

  You: omg this is really working isint it?  
  ...

  \[EIRENE | risk=0.0 | mood=0.13\]

  Athena: Absolutely, I'm designed to help you in a gentle, understanding way. Your interaction is a testament to our effective collaboration. Let's continue exploring together.

  Mode: EIRENE  |  Mood: 0.13  |  Surplus: 0.0  |  Harmony: \-2.065

  Commands:  
      1\. Start session  
      2\. Send message  
      3\. End session  
      4\. View system status  
      5\. View memory  
      6\. Add open thread  
      7\. Record lens-shift  
      8\. View ABML table  
      9\. View audit log  
     10\. Export all user data  
     11\. Forget all memory  \[\!\]  
     12\. Toggle sleep mode  
     13\. Toggle harmony guardian  
     14\. Reset Guardian (human override)  
     15\. Test crisis / Aegis response  
      0\. Exit

  Command: 

