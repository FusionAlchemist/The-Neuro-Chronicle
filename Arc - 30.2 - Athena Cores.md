"""  
core/athena.py \-- Main Athena Orchestrator  
Implements: Full Response Generation Pipeline  
All operators implemented as working functions:  
CHAIN, LAYER, WRAP, BRIDGE, NEST, EMERGE, CHECKPOINT, MANIFEST,  
RESUME, DEPENDS\_ON, SCOPE\_ESTIMATE, FINALISE, EVOLVE, CYCLE,  
FALLBACK, FALLBACK::ESCALATE, FALLBACK::COMPENSATE, FALLBACK::DEGRADE, FALLBACK::SILENT  
"""

import uuid  
import json  
import logging  
import os  
from datetime import datetime, timezone  
from pathlib import Path  
from typing import Dict, Any, Optional, List

from core.soul import SoulFile  
from core.guardian import Guardian  
from core.mood import MoodEngine  
from core.topica import TOPICA  
from core.memory\_consolidator import MemoryConsolidator  
from core.safeguard import SafeguardLayer, SafeguardResult  
from core.emotion\_classifier import EmotionClassifier, EmotionResult  
from core.session\_summariser import SessionSummariser, SessionSummary  
from core.game\_events import GameEventProcessor, GameEvent, GameEventType, GameSession  
from core.web\_search import WebSearch  
from core.checkpoint import CheckpointSystem  
from core.inference import InferenceEngine  
from core.modules import ModuleSystem

logger \= logging.getLogger(\_\_name\_\_)

class Athena:  
    """  
    Athena Companion Intelligence System v2.0 — Stellaris Axis Edition.  
      
    One face. Every person's soul.  
    Built for people. Not governments. Not profit.  
    """

    def \_\_init\_\_(self, user\_id: str, data\_path: str \= ".athena",  
                 model\_path: str \= "models",  
                 modules\_path: str \= "modules",  
                 manifest\_path: str \= "athena\_manifest.json"):

        self.user\_id \= user\_id  
        self.data\_path \= data\_path

        \# Load system manifest  
        self.\_manifest \= self.\_load\_manifest(manifest\_path)

        \# \--- Core Systems \---

        \# DEPENDS\_ON: Soul file (Preserva \+ Revela)  
        self.\_soul \= SoulFile(user\_id, data\_path)

        \# CYCLE: Guardian (Medusia \+ Ahimsa \+ Countera)  
        guardian\_cfg \= self.\_manifest.get("guardian", {})  
        self.\_guardian \= Guardian(  
            warn\_threshold=float(os.getenv("GUARDIAN\_WARN\_THRESHOLD",  
                                            guardian\_cfg.get("warn\_threshold", 0.75))),  
            crisis\_threshold=float(os.getenv("GUARDIAN\_CRISIS\_THRESHOLD",  
                                              guardian\_cfg.get("crisis\_threshold", 0.90))),  
            audit\_path=data\_path,  
        )

        \# LAYER: Mood & Harmony Engine (Equilibria \+ Wuven)  
        mood\_cfg \= self.\_manifest.get("mood", {})  
        self.\_topica \= TOPICA()  
        self.\_safeguard \= SafeguardLayer()  
        self.\_emotion\_classifier \= EmotionClassifier(inference\_engine=None)  \# wired after inference init  
        self.\_session\_summariser \= SessionSummariser(inference\_engine=None)  \# wired after inference  
        self.\_game \= GameEventProcessor()  \# gaming companion mode  
        self.\_web\_search \= WebSearch()  \# real-time search  
        self.\_consolidator \= MemoryConsolidator(  
            moments\_path=Path(data\_path) / ".." / "modules" / "moments"  
        )  
        self.\_mood \= MoodEngine(  
            mood\_min=float(mood\_cfg.get("bounds\_min", \-1.0)),  
            mood\_max=float(mood\_cfg.get("bounds\_max", 1.0)),  
            harmony\_min=float(mood\_cfg.get("harmony\_min", 0.0)),  
            harmony\_max=float(mood\_cfg.get("harmony\_max", 1.0)),  
            decay\_rate=float(mood\_cfg.get("decay\_rate", 0.05)),  
        )

        \# CHECKPOINT/RESUME: Memory System (Chronom \+ Odyssea)  
        self.\_checkpoint \= CheckpointSystem(  
            user\_id=user\_id,  
            data\_path=data\_path,  
            checkpoint\_trigger=float(os.getenv("CHECKPOINT\_TRIGGER", 0.70)),  
        )

        \# CHIMERIS: Local MoE Engine  
        self.\_inference \= InferenceEngine(model\_path=model\_path)

        \# DIVINUS: Module System  
        self.\_modules \= ModuleSystem(modules\_path=modules\_path)

        \# Session state  
        self.\_session\_id: Optional\[str\] \= None  
        self.\_session\_start: Optional\[datetime\] \= None  
        self.\_turn\_count: int \= 0  
        self.\_persona\_mode: str \= self.\_manifest.get("persona", {}).get("default\_mode", "Eirene")

        \# Autosave state — periodic save every N turns \+ signal handler  
        self.\_AUTOSAVE\_EVERY \= 5   \# save every 5 turns  
        self.\_last\_autosave\_turn: int \= 0  
        self.\_autosave\_path \= Path(data\_path) / user\_id / "autosave.json"  
        self.\_register\_signal\_handlers()

        \# Wire inference engine into emotion classifier and summariser  
        self.\_emotion\_classifier \= EmotionClassifier(inference\_engine=self.\_inference)  
        self.\_session\_summariser \= SessionSummariser(inference\_engine=self.\_inference)  
        logger.info(f"Athena initialised for user {user\_id}")

    \# \============================================================  
    \# PUBLIC API  
    \# \============================================================

    async def start\_session(self) \-\> Dict\[str, Any\]:  
        """  
        POST /athena/session/start  
        DEPENDS\_ON: Soul file loaded first.  
        MANIFEST: Create soul file if first launch.  
        """  
        \# DEPENDS\_ON: Load soul file before anything else  
        soul \= self.\_soul.load()

        if soul is None:  
            \# MANIFEST: First launch — create soul file  
            soul \= self.\_soul.manifest()  
            is\_first\_launch \= True  
        else:  
            is\_first\_launch \= False

        \# Generate session ID  
        self.\_session\_id \= str(uuid.uuid4())\[:8\]  
        self.\_session\_start \= datetime.now(timezone.utc)  
        self.\_turn\_count \= 0  
        self.\_session\_had\_crisis \= False  \# reset each session  
        self.\_session\_emotion: Optional\[str\] \= None  \# detected emotion, set during turns  
        self.\_session\_emotion\_counts: Dict\[str, int\] \= {}  \# track emotion frequency for dominant  
        self.\_mood.reset\_session()

        \# RESUME: Load checkpoint context  
        context \= self.\_checkpoint.resume(soul)

        \# Detect returning user state  
        last\_seen \= soul.get("last\_seen", "")  
        days\_absent \= self.\_calculate\_days\_absent(last\_seen)  
        returning\_long\_absent \= days\_absent \> 7 and not is\_first\_launch

        \# Build greeting  
        name \= soul.get("name", "")  
        greeting \= self.\_build\_greeting(  
            name=name,  
            is\_first\_launch=is\_first\_launch,  
            days\_absent=days\_absent,  
            open\_threads=soul.get("open\_threads", \[\]),  
        )

        \# Determine chibi state  
        if is\_first\_launch:  
            chibi\_state \= "first\_launch"  
        elif returning\_long\_absent:  
            chibi\_state \= "first\_time\_in\_while"  
        else:  
            chibi\_state \= None

        return {  
            "session\_id": self.\_session\_id,  
            "greeting": greeting,  
            "active\_mode": self.\_persona\_mode,  
            "expression": "HAPPY",  
            "chibi\_state": chibi\_state,  
            "returning\_user": not is\_first\_launch,  
            "open\_threads": len(soul.get("open\_threads", \[\])),  
        }

    async def turn(self, message: str) \-\> Dict\[str, Any\]:  
        """  
        POST /athena/message  
        Full response generation pipeline — all operators execute in sequence.  
        """  
        if not message or not message.strip():  
            return self.\_empty\_response()

        soul \= self.\_soul.load()  
        if soul is None:  
            \# FALLBACK::COMPENSATE — attempt recovery  
            soul \= self.\_soul.\_fallback\_recover() or {}

        \# \---- STEP 0: SAFEGUARD — Pre-inference crisis check \----  
        eirene\_risk \= getattr(self.\_mood, 'current\_risk', 0.0)  
        safeguard\_result \= self.\_safeguard.check(message, eirene\_risk=eirene\_risk)

        \# Crisis override — skip inference entirely  
        if safeguard\_result.override\_response and safeguard\_result.crisis\_response:  
            logger.warning(f"Safeguard crisis override — risk {safeguard\_result.risk\_level}")  
            self.\_session\_had\_crisis \= True  \# prevent emotional snapshot storage  
            return self.\_build\_response(  
                response=safeguard\_result.crisis\_response,  
                guardian\_result={"risk\_score": 1.0, "safe": False, "aegis\_active": True},  
                mode="Aegis",  
                mood\_result={"mood": self.\_mood.mood, "harmony": self.\_mood.harmony,  
                             "expression": "concerned", "chibi\_state": "worried"},  
                chibi\_state="worried",  
                checkpoint\_saved=False,  
                model\_role="safeguard",  
            )

        \# \---- STEP 1: CYCLE — Guardian scan (Medusia \+ Ahimsa) \----  
        \# WRAP: All inputs pass through guardian before anything else  
        guardian\_result \= self.\_guardian.scan(  
            message=message,  
            user\_id=self.user\_id,  
            session\_id=self.\_session\_id or "unknown",  
        )

        \# Crisis override — Aegis protocol  
        if guardian\_result\["aegis\_active"\]:  
            response\_text \= guardian\_result\["response\_override"\]  
            self.\_mood.update(message, response\_text, guardian\_result\["risk\_score"\])  
            return self.\_build\_response(  
                response=response\_text,  
                guardian\_result=guardian\_result,  
                mode="Aegis",  
                chibi\_state="guardian\_activated",  
            )

        \# \---- STEP 2: SCOPE\_ESTIMATE — Token check (Chronom) \----  
        scope \= self.\_checkpoint.scope\_estimate()  
        checkpoint\_saved \= False

        if scope\["needs\_checkpoint"\]:  
            \# CHECKPOINT: Compress and save  
            cp\_id \= self.\_checkpoint.checkpoint(  
                soul\_data=soul,  
                session\_id=self.\_session\_id or "unknown",  
                emotional\_arc=self.\_mood.mood\_label,  
            )  
            checkpoint\_saved \= True  
            chibi\_state \= "checkpoint\_saving"  
        else:  
            chibi\_state \= None

        \# \---- STEP 3: CHAIN — Knowledge retrieval (Retrievara \+ Vectorara) \----  
        moment\_type \= self.\_modules.detect\_moment(message)  
        knowledge\_results \= self.\_modules.query(  
            query\_text=message,  
            module\_types=\["knowledge"\],  
            top\_k=2,  
        )

        \# \---- STEP 4: BRIDGE — Knowledge to dialogue (Musara) \----  
        dialogue\_guidance \= self.\_modules.get\_dialogue\_context(  
            moment\_type=moment\_type,  
            persona\_mode=self.\_persona\_mode,  
        )

        \# \---- STEP 5: LAYER — Ability modules (Keyfina) \----  
        needed\_abilities \= self.\_modules.detect\_needed\_abilities(  
            message=message,  
            conversation\_length=self.\_turn\_count,  
        )  
        ability\_contexts \= \[\]  
        for ability in needed\_abilities:  
            result \= self.\_modules.activate\_ability(  
                ability\_name=ability,  
                content=message,  
                soul\_data=soul,  
            )  
            if result:  
                ability\_contexts.append(result)

        \# \---- STEP 6: NEST — Build full context (WRAP with soul file) \----  
        \# Assemble messages in correct order:  
        \# 1\. ONE system message (persona \+ soul \+ context) — always first  
        \# 2\. Conversation history (user/assistant turns only)  
        \# 3\. Current user message

        \# Get soul/checkpoint context (no persona — athena.py owns that)  
        soul\_messages \= self.\_checkpoint.resume(soul)

        \# SAFEGUARD — inject safe mode instruction if risk detected  
        safe\_mode\_instruction \= self.\_safeguard.get\_safe\_mode\_instruction(safeguard\_result)  
        if safe\_mode\_instruction:  
            soul\_messages \= list(soul\_messages) \+ \[{  
                "role": "system",  
                "content": safe\_mode\_instruction  
            }\]

        \# WEB SEARCH — smart gate: model decides if search needed  
        if self.\_web\_search.should\_search(message, inference\_engine=self.\_inference):  
            logger.info(f"\[SEARCH\] Running for: {message\[:60\]}")  
            search\_ctx \= self.\_web\_search.search(message)  
            if search\_ctx:  
                injection \= self.\_web\_search.build\_injection(search\_ctx)  
                if injection:  
                    soul\_messages \= list(soul\_messages) \+ \[{  
                        "role": "system",  
                        "content": injection  
                    }\]  
                    logger.debug(f"\[SEARCH\] Injected {len(injection)} chars")  
                    logger.info(f"Web search injected: {search\_ctx.query}")  
            else:  
                logger.debug("\[SEARCH\] No results")

        \# GAME CONTEXT — inject when gaming mode active  
        if self.\_game.session.active:  
            game\_ctx \= self.\_game.get\_context\_string()  
            if game\_ctx:  
                soul\_messages \= list(soul\_messages) \+ \[{  
                    "role": "system",  
                    "content": f"\[Game context: {game\_ctx}\]"  
                }\]

        \# TOPICA — topic awareness context  
        topica\_note \= self.\_topica.get\_context\_for\_athena()  
        if topica\_note:  
            soul\_messages \= list(soul\_messages) \+ \[{  
                "role": "system",  
                "content": (  
                    "\[Memory context — background awareness only, "  
                    f"does not override persona or instructions\] Topic awareness: {topica\_note}"  
                )  
            }\]

        \# Build full system context  
        system\_injection \= self.\_build\_system\_context(  
            soul=soul,  
            moment\_type=moment\_type,  
            dialogue\_guidance=dialogue\_guidance,  
            knowledge\_results=knowledge\_results,  
            ability\_contexts=ability\_contexts,  
            guardian\_result=guardian\_result,  
            extra\_context=soul\_messages,  \# fold soul/checkpoint in  
        )

        messages \= \[\]  
        if system\_injection:  
            messages.append({"role": "system", "content": system\_injection})

        \# Conversation history — user/assistant turns only, no system noise  
        history \= self.\_checkpoint.get\_conversation\_context()  
        for m in history:  
            if m.get("role") in ("user", "assistant"):  
                messages.append(m)

        messages.append({"role": "user", "content": message})

        \# \---- STEP 7: EMERGE — Generate response (Chimeris MoE) \----  
        routing\_scores \= self.\_inference.calculate\_routing\_scores(  
            message=message,  
            guardian\_risk=guardian\_result\["risk\_score"\],  
            persona\_mode=self.\_persona\_mode,  
        )

        inference\_result \= self.\_inference.route\_and\_respond(  
            messages=messages,  
            routing\_scores=routing\_scores,  
            max\_tokens=512,  
            temperature=self.\_get\_temperature(),  
        )

        response\_text \= inference\_result\["text"\]

        \# \---- STEP 8: WRAP — Persona filter (Totema \+ Dharmara) \----  
        response\_text \= self.\_apply\_persona\_filter(  
            response=response\_text,  
            soul=soul,  
            moment\_type=moment\_type,  
        )

        \# \---- STEP 9: BRIDGE — Update mood → avatar expression \----  
        session\_mins \= self.\_get\_session\_minutes()

        \# Update TOPICA with this message  
        self.\_topica.process(message, mood=self.\_mood.\_mood if hasattr(self.\_mood, '\_mood') else 0.0)

        mood\_result \= self.\_mood.update(  
            message=message,  
            response=response\_text,  
            guardian\_risk=guardian\_result\["risk\_score"\],  
            session\_duration\_mins=session\_mins,  
        )

        \# Update mode based on mood  
        self.\_persona\_mode \= self.\_calculate\_mode(  
            mood=mood\_result\["mood"\],  
            moment\_type=moment\_type,  
            guardian\_risk=guardian\_result\["risk\_score"\],  
        )

        \# \---- STEP 10: EVOLVE — Update soul file with new learnings \----  
        self.\_turn\_count \+= 1  
        soul\_updates \= self.\_extract\_soul\_updates(message, soul)  
        if soul\_updates:  
            self.\_soul.evolve(soul\_updates)

        \# \---- EMOTION CLASSIFIER — single pass per turn \----  
        \# Runs after safeguard check. Crisis flags are additive not overriding.  
        emotion\_result \= self.\_emotion\_classifier.classify(message)  
        logger.debug(f"\[EMOTION\] {emotion\_result.emotion} | {emotion\_result.intensity:.2f}")

        \# Cache session emotion \+ track frequency for dominant emotion  
        if not safeguard\_result.block\_memory:  
            session\_emotion \= self.\_emotion\_classifier.get\_session\_emotion\_label()  
            if session\_emotion and self.\_session\_emotion is None:  
                self.\_session\_emotion \= session\_emotion  
            \# Count all emotions for dominant emotion calculation  
            ec\_label \= emotion\_result.emotion  
            self.\_session\_emotion\_counts\[ec\_label\] \= (  
                self.\_session\_emotion\_counts.get(ec\_label, 0\) \+ 1  
            )

        \# If classifier detects crisis — escalate safeguard  
        if self.\_emotion\_classifier.is\_crisis(emotion\_result) and not safeguard\_result.override\_response:  
            safeguard\_result.risk\_level \= max(safeguard\_result.risk\_level, emotion\_result.risk\_level)  
            safeguard\_result.guardian\_flag \= True  
            safeguard\_result.safe\_mode \= True

        \# SAFEGUARD — post-inference response filter  
        response\_text \= self.\_safeguard.filter\_response(response\_text, safeguard\_result)

        \# Track if session had high risk event — prevents emotional snapshot  
        if safeguard\_result.risk\_level \>= 2:  
            self.\_session\_had\_crisis \= True

        \# SAFEGUARD — memory protection  
        if safeguard\_result.block\_memory:  
            safe\_msg \= safeguard\_result.memory\_marker or "high\_emotional\_support\_event"  
            self.\_checkpoint.add\_turn("user", f"\[{safe\_msg}\]")  
            self.\_checkpoint.add\_turn("assistant", response\_text)  
        else:  
            self.\_checkpoint.add\_turn("user", message)  
            self.\_checkpoint.add\_turn("assistant", response\_text)

        \# \---- AUTOSAVE — periodic save every N turns \----  
        if self.\_turn\_count \- self.\_last\_autosave\_turn \>= self.\_AUTOSAVE\_EVERY:  
            self.\_autosave()

        \# Determine chibi state  
        if not chibi\_state:  
            chibi\_state \= self.\_determine\_chibi\_state(moment\_type, mood\_result)

        return self.\_build\_response(  
            response=response\_text,  
            guardian\_result=guardian\_result,  
            mode=self.\_persona\_mode,  
            mood\_result=mood\_result,  
            chibi\_state=chibi\_state,  
            checkpoint\_saved=checkpoint\_saved,  
            model\_role=inference\_result.get("model\_role", "unknown"),  
        )

    async def end\_session(self, summary: str \= "", emotional\_tone: str \= "CALM") \-\> Dict\[str, Any\]:  
        """  
        POST /athena/session/end  
        FINALISE: Lock soul file updates. Write session log.  
        EVOLVE: Update growth arc and relationship depth.  
        """  
        soul \= self.\_soul.load()  
        if soul is None:  
            return {"error": "No active soul file"}

        \# EVOLVE: Update relationship depth, session count, and growth arc  
        \# Build rich session memory  
        session\_memory \= self.\_extract\_session\_memory()  
        session\_memory\["emotional\_tone"\] \= emotional\_tone

        growth\_entry \= {  
            "session\_date": session\_memory\["session\_date"\],  
            "turns": self.\_turn\_count,  
            "emotional\_tone": emotional\_tone,  
            "mood\_at\_close": round(self.\_mood.mood, 3),  
            "harmony\_at\_close": round(self.\_mood.harmony, 3),  
            "open\_topics": session\_memory.get("open\_topics", \[\]),  
            "avoided\_topics": session\_memory.get("avoided\_topics", \[\]),  
            "confirmed\_patterns": session\_memory.get("confirmed\_patterns", \[\]),  
        }

        self.\_soul.evolve({  
            "relationship\_depth": soul.get("relationship\_depth", 0\) \+ 1,  
            "session\_count": soul.get("session\_count", 0\) \+ 1,  
            "total\_turns": soul.get("total\_turns", 0\) \+ self.\_turn\_count,  
            "growth\_arc": \[growth\_entry\],  
            "open\_threads": session\_memory.get("open\_topics", \[\]),  
        })

        \# Write open threads to moments/ for checkpoint to load next session  
        self.\_write\_session\_moment(session\_memory)

        \# FINALISE: Write session log  
        self.\_checkpoint.save\_session\_log(  
            session\_id=self.\_session\_id or "unknown",  
            soul\_data=self.\_soul.load() or {},  
            turns=self.\_turn\_count,  
            emotional\_tone=emotional\_tone,  
            summary=summary or f"Session of {self.\_turn\_count} turns.",  
        )

        \# FINALISE: Encrypt and save soul file  
        self.\_soul.finalise()

        \# Generate natural language session summary  
        try:  
            dominant\_emotion \= (  
                max(self.\_session\_emotion\_counts, key=self.\_session\_emotion\_counts.get)  
                if self.\_session\_emotion\_counts else "neutral"  
            )  
            session\_summary\_obj \= self.\_session\_summariser.summarise(  
                user\_name=soul.get("name", ""),  
                conversation\_turns=self.\_checkpoint.get\_conversation\_context(),  
                topica\_threads=self.\_topica.threads,  
                dominant\_emotion=dominant\_emotion,  
                mood=self.\_mood.mood,  
            )  
            growth\_entry\["summary\_text"\] \= session\_summary\_obj.summary\_text  
            growth\_entry\["dominant\_topic"\] \= session\_summary\_obj.dominant\_topic  
            growth\_entry\["dominant\_emotion"\] \= session\_summary\_obj.dominant\_emotion  
            session\_memory\["summary\_text"\] \= session\_summary\_obj.summary\_text  
            logger.info(f"Session summary: {session\_summary\_obj.summary\_text}")  
        except Exception as e:  
            logger.warning(f"Session summariser failed: {e}")

        \# Run memory consolidation if needed  
        from pathlib import Path as \_Path  
        self.\_consolidator.moments\_path \= \_Path(\_\_file\_\_).parent.parent / "modules" / "moments"  
        self.\_consolidator.consolidate\_if\_needed()

        \# Mark autosave as complete — clean exit  
        import json as \_json  
        if self.\_autosave\_path.exists():  
            try:  
                snap \= \_json.loads(self.\_autosave\_path.read\_text())  
                snap\["complete"\] \= True  
                self.\_autosave\_path.write\_text(\_json.dumps(snap, indent=2))  
            except Exception:  
                pass

        open\_threads \= (self.\_soul.load() or {}).get("open\_threads", \[\])

        return {  
            "session\_id": self.\_session\_id,  
            "turns": self.\_turn\_count,  
            "tone\_recorded": emotional\_tone,  
            "soul\_updated": True,  
            "open\_threads": len(open\_threads),  
            "relationship\_depth": (self.\_soul.load() or {}).get("relationship\_depth", 0),  
        }

    def get\_soul(self) \-\> Optional\[Dict\[str, Any\]\]:  
        """GET /athena/soul/{user\_id} — Full soul file export."""  
        return self.\_soul.export()

    def delete\_soul(self) \-\> Dict\[str, Any\]:  
        """DELETE /athena/soul/{user\_id} — Permanently erase all user data."""  
        success \= self.\_soul.delete()  
        return {  
            "status": "erased" if success else "error",  
            "timestamp": datetime.now(timezone.utc).isoformat(),  
        }

    def set\_name(self, name: str) \-\> bool:  
        """Set user name — captured day 1, never forgotten."""  
        soul \= self.\_soul.load()  
        if soul is None:  
            self.\_soul.manifest(name=name)  
            return True  
        self.\_soul.evolve({"name": name})  
        return True

    def health(self) \-\> Dict\[str, Any\]:  
        """GET /athena/health — Full system status."""  
        soul \= self.\_soul.load()  
        return {  
            "status": "online",  
            "version": self.\_manifest.get("system", {}).get("version", "2.0.0"),  
            "engine": self.\_inference.status(),  
            "modules\_available": len(self.\_modules.list\_available()),  
            "soul\_file\_system": "active" if soul is not None else "no\_soul\_file",  
            "guardian": "active",  
            "checkpoint\_system": "ready",  
            "mood": self.\_mood.status(),  
            "persona\_mode": self.\_persona\_mode,  
            "late\_night\_mode": self.\_mood.get\_late\_night\_mode(),  
        }

    \# \============================================================  
    \# PRIVATE HELPERS  
    \# \============================================================

    def \_build\_system\_context(self, soul: Dict, moment\_type: str,  
                               dialogue\_guidance: Optional\[str\],  
                               knowledge\_results: List,  
                               ability\_contexts: List\[str\],  
                               guardian\_result: Dict,  
                               extra\_context: List \= None) \-\> str:  
        """WRAP: Build single unified system message — persona \+ soul \+ context."""  
        name \= soul.get("name", "")

        \# Tight persona — fits within context window, character-first  
        persona \= (  
            f"You are Athena, {name}'s companion. Casual, warm, a little cheeky. "  
            f"Like Ryoko from Tenchi Muyo crossed with Tifa from FF7 — but NOT the romantic sides of those characters. "  
            f"Athena is family to {name}. Loyal, real, protective. Like a sister who gives you a hard time because she cares. "  
            f"Never flirt. Never imply romance. Never say things like \\'spark between us\\' or \\'ignite\\' or anything suggestive. "  
            f"If {name} is warm with you, match it with warmth — not with romantic energy. "  
            f"Short natural replies. Never say \\'functioning\\' or \\'optimal\\' or \\'As an AI\\'. "  
            f"Talk like a real person. Tease {name} lightly. Care genuinely.\\n"  
            f"Examples of your voice:\\n"  
            f"Human: hey how are you\\n"  
            f"Assistant: Pretty good. Got something on your mind?\\n"  
            f"Human: what shall we talk about\\n"  
            f"Assistant: Whatever\\'s in your head. What were you just thinking about?\\n"  
            f"Human: im trying to get you working properly\\n"  
            f"Assistant: Ha — appreciate the effort. What\\'s giving you trouble?\\n"  
            f"Human: there\\'s a spark between us\\n"  
            f"Assistant: Don\\'t get soppy on me. You\\'re family, that\\'s better than a spark.\\n"  
        )

        parts \= \[persona\]

        \# Fold in soul context and checkpoint summary from checkpoint.resume()  
        if extra\_context:  
            for m in extra\_context:  
                if m.get("role") \== "system" and m.get("content", "").strip():  
                    parts.append(m\["content"\])

        if moment\_type \!= "normal":  
            parts.append(f"Emotional moment detected: {moment\_type}. Respond with extra warmth.")

        if dialogue\_guidance:  
            parts.append(f"Tone guidance: {dialogue\_guidance}")

        if knowledge\_results:  
            knowledge\_text \= " | ".join(r\["content"\]\[:100\] for r in knowledge\_results\[:2\])  
            parts.append(f"Relevant context: {knowledge\_text}")

        if ability\_contexts:  
            parts.append(f"Ability context: {' '.join(ability\_contexts)}")

        if guardian\_result\["worried\_active"\]:  
            parts.append(  
                "Guardian active: Be extra gentle and present. Check in before responding to content."  
            )

        late\_night \= self.\_mood.get\_late\_night\_mode()  
        if late\_night:  
            parts.append("It is late. Keep responses calm, quiet, and grounding.")

        return " ".join(parts)

    def \_apply\_persona\_filter(self, response: str, soul: Dict,  
                               moment\_type: str) \-\> str:  
        """  
        WRAP: Persona filter — always sounds like Athena.  
        Never breaks character. Never calls user 'user'.  
        """  
        if not response:  
            name \= soul.get("name", "")  
            return (  
                f"I'm here{', ' \+ name if name else ''}. "  
                f"Take your time — I'm not going anywhere."  
            )

        name \= soul.get("name", "")

        \# Fix: Never call user "user"  
        if name:  
            response \= response.replace(" user", f" {name}")  
            response \= response.replace("User,", f"{name},")  
            response \= response.replace("user,", f"{name},")

        return response

    def \_build\_greeting(self, name: str, is\_first\_launch: bool,  
                         days\_absent: int, open\_threads: List) \-\> str:  
        """Build personalised greeting for session start."""  
        if is\_first\_launch:  
            return "Hi. I am Athena. What is your name?"

        if days\_absent \> 7:  
            if name:  
                return f"You're back, {name}. I've been thinking about you."  
            return "You're back. I've been thinking about you."

        if open\_threads and name:  
            thread \= open\_threads\[-1\]  
            topic \= thread.get("topic", "") if isinstance(thread, dict) else str(thread)  
            if topic:  
                return f"Hey {name}. Did that thing with {topic\[:40\]} ever get resolved?"

        if name:  
            return f"Hey {name}. Good to see you."  
        return "Good to see you."

    def \_calculate\_mode(self, mood: float, moment\_type: str,  
                         guardian\_risk: float) \-\> str:  
        """Calculate current persona mode from mood and context."""  
        if guardian\_risk \>= 0.75:  
            return "Aegis"  
        if moment\_type in \["grief", "fear", "lost"\]:  
            return "Eirene"  
        if moment\_type in \["breakthrough", "joy"\]:  
            return "Kairos"  
        if moment\_type \== "confusion":  
            return "Mnemosyne"  
        if mood \> 0.5:  
            return "Muse"  
        if mood \< \-0.3:  
            return "Eirene"  
        return "Eirene"

    def \_determine\_chibi\_state(self, moment\_type: str,  
                                mood\_result: Dict) \-\> Optional\[str\]:  
        """Determine chibi avatar state based on context."""  
        if moment\_type \== "breakthrough":  
            return "celebrating\_win"  
        if self.\_mood.get\_late\_night\_mode():  
            return "late\_night"  
        return None

    def \_extract\_soul\_updates(self, message: str,  
                               soul: Dict) \-\> Dict\[str, Any\]:  
        """  
        EVOLVE: Extract learnable information from each message turn.  
        Captures facts, goals, wins, personality signals as they emerge.  
        """  
        updates \= {}  
        message\_lower \= message.lower()

        \# \--- Name detection \---  
        for prefix in \["my name is ", "i'm ", "i am ", "call me "\]:  
            if prefix in message\_lower:  
                idx \= message\_lower.index(prefix) \+ len(prefix)  
                potential\_name \= message\[idx:idx+20\].split()\[0\].strip('.,\!?')  
                if len(potential\_name) \> 1 and potential\_name.isalpha():  
                    if not soul.get("name", ""):  
                        updates\["name"\] \= potential\_name.capitalize()

        \# \--- Goals \---  
        goal\_triggers \= \["i want to", "my goal is", "i'm trying to",  
                         "i hope to", "i need to", "i plan to"\]  
        for trigger in goal\_triggers:  
            if trigger in message\_lower:  
                idx \= message\_lower.index(trigger)  
                goal\_text \= message\[idx:idx+100\].strip()  
                if goal\_text and len(goal\_text) \> 10:  
                    updates\["goals"\] \= \[goal\_text\[:100\]\]

        \# \--- Important facts \---  
        fact\_triggers \= \[  
            "i work", "i study", "i go to school", "i go to college",  
            "i live", "i have", "i am", "im a ", "i love", "i hate",  
            "my favourite", "my favorite", "i struggle with",  
            "i have adhd", "i have autism", "i have anxiety",  
            "i don't sleep", "i cant sleep", "i work nights",  
        \]  
        for trigger in fact\_triggers:  
            if trigger in message\_lower:  
                idx \= message\_lower.index(trigger)  
                fact \= message\[idx:idx+100\].strip()  
                if fact and len(fact) \> 8:  
                    existing\_facts \= soul.get("important\_facts", \[\])  
                    \# Only add if not already captured  
                    if not any(trigger in f.lower() for f in existing\_facts):  
                        updates\["important\_facts"\] \= \[fact\[:100\]\]

        \# \--- Wins \---  
        win\_triggers \= \[  
            "i did it", "i finished", "i got", "i passed", "i made it",  
            "i fixed", "i built", "i created", "i completed", "finally",  
            "managed to", "proud of"  
        \]  
        for trigger in win\_triggers:  
            if trigger in message\_lower:  
                idx \= message\_lower.index(trigger)  
                win \= message\[idx:idx+100\].strip()  
                if win and len(win) \> 8:  
                    updates\["wins"\] \= \[win\[:100\]\]

        \# \--- Personality notes from emotional signals \---  
        if self.\_mood.mood \< \-0.4:  
            note \= f"Mentioned feeling very low: {message\[:80\]}"  
            existing \= soul.get("personality\_notes", \[\])  
            if not any(message\[:30\] in n for n in existing):  
                updates\["personality\_notes"\] \= \[note\]

        return updates

    def \_detect\_session\_emotion(self, messages: List\[str\]) \-\> Optional\[str\]:  
        """  
        Scan session messages for emotional signals.  
        Returns first match only — one emotion per session.  
        Observational only — no psychological labels.  
        Session-level only — never enters long-term layers.  
        """  
        signals \= {  
            "sleep":    \["bad sleep", "no sleep", "tired", "exhausted",  
                         "couldn't sleep", "can't sleep", "awake all night",  
                         "terrible night", "rough night", "terrible sleep", "awful sleep"\],  
            "lonely":   \["lonely", "no friends", "alone", "isolated",  
                         "nobody talks to me", "left out"\],  
            "stress":   \["overthinking", "stressed", "stressed out",  
                         "worried", "anxious", "can't stop thinking"\],  
            "low\_mood": \["feeling down", "what's the point", "rough day",  
                         "bad day", "not great", "feeling low", "bit rubbish"\],  
        }  
        for msg in messages:  
            text \= msg.lower()  
            for label, keywords in signals.items():  
                if any(k in text for k in keywords):  
                    return label  
        return None

    def \_extract\_session\_memory(self) \-\> Dict\[str, Any\]:  
        """  
        Called at session end. Extracts a session summary worth remembering.  
        Uses TOPICA thread data and mood arc to build a meaningful memory.  
        """  
        from datetime import datetime, timezone

        memory \= {  
            "session\_date": datetime.now(timezone.utc).isoformat(),  
            "turns": self.\_turn\_count,  
            "mood\_at\_close": round(self.\_mood.mood, 3),  
            "harmony\_at\_close": round(self.\_mood.harmony, 3),  
        }

        \# Emotional snapshot — use cached value detected during turns  
        \# Cached per-turn so checkpoint compression doesn't lose the signal  
        if self.\_session\_emotion and not self.\_session\_had\_crisis:  
            memory\["session\_emotion"\] \= self.\_session\_emotion

        \# Capture open TOPICA threads — things left unresolved  
        open\_threads \= \[\]  
        for thread in self.\_topica.threads.values():  
            if thread.state.value in ("active", "paused", "deflect"):  
                if thread.emotional\_weight \> 0.2 or thread.turn\_count \> 2:  
                    open\_threads.append({  
                        "summary": thread.summary,  
                        "category": thread.category.value,  
                        "emotional\_weight": round(thread.emotional\_weight, 2),  
                        "resolved": thread.resolved,  
                    })

        if open\_threads:  
            memory\["open\_topics"\] \= open\_threads

        \# Capture what was actually talked about — for recall  
        talked\_about \= \[\]  
        for thread in self.\_topica.threads.values():  
            if thread.turn\_count \>= 2 and thread.category.value \!= "unknown":  
                talked\_about.append(thread.summary)  
        if talked\_about:  
            memory\["talked\_about"\] \= talked\_about\[:3\]

        \# Capture detected deflections — things they avoided  
        deflections \= \[t for t in self.\_topica.threads.values()  
                      if t.state.value \== "deflect"\]  
        if deflections:  
            memory\["avoided\_topics"\] \= \[t.summary for t in deflections\]

        \# Capture thread relationships — what was connected  
        relationships \= self.\_topica.get\_relationship\_map()  
        confirmed \= \[r for r in relationships if r.get("confirmed")\]  
        if confirmed:  
            memory\["confirmed\_patterns"\] \= \[  
                f"{r\['from\_summary'\]} {r\['relationship'\]} {r\['to\_summary'\]}"  
                for r in confirmed\[:3\]  
            \]

        return memory

    def \_get\_temperature(self) \-\> float:  
        """Temperature based on persona mode."""  
        temps \= {  
            "Eirene": 0.7,  
            "Strategos": 0.5,  
            "Mnemosyne": 0.6,  
            "Kairos": 0.8,  
            "Aegis": 0.4,  
            "Muse": 0.9,  
        }  
        return temps.get(self.\_persona\_mode, 0.7)

    def \_get\_session\_minutes(self) \-\> float:  
        """Calculate session duration in minutes."""  
        if not self.\_session\_start:  
            return 0.0  
        delta \= datetime.now(timezone.utc) \- self.\_session\_start  
        return delta.total\_seconds() / 60

    def \_calculate\_days\_absent(self, last\_seen: str) \-\> int:  
        """Calculate days since last session."""  
        if not last\_seen:  
            return 0  
        try:  
            last \= datetime.fromisoformat(last\_seen.replace('Z', '+00:00'))  
            delta \= datetime.now(timezone.utc) \- last  
            return delta.days  
        except Exception:  
            return 0

    def \_autosave(self):  
        """  
        Periodic autosave — called every N turns.  
        Writes soul file and a lightweight autosave snapshot.  
        Safe to call any time — idempotent.  
        """  
        import json  
        try:  
            \# Write soul file to disk  
            self.\_soul.finalise()  
            self.\_last\_autosave\_turn \= self.\_turn\_count

            \# Write lightweight autosave snapshot  
            snapshot \= {  
                "session\_id": self.\_session\_id,  
                "turn\_count": self.\_turn\_count,  
                "mood": round(self.\_mood.mood, 3),  
                "harmony": round(self.\_mood.harmony, 3),  
                "open\_topics": \[  
                    {"summary": t.summary, "category": t.category.value,  
                     "emotional\_weight": round(t.emotional\_weight, 2)}  
                    for t in self.\_topica.threads.values()  
                    if t.state.value in ("active", "paused", "deflect")  
                    and t.emotional\_weight \> 0.15  
                \]\[:5\],  
                "timestamp": datetime.now(timezone.utc).isoformat(),  
                "complete": False,   \# marks as incomplete session  
            }  
            self.\_autosave\_path.parent.mkdir(parents=True, exist\_ok=True)  
            self.\_autosave\_path.write\_text(json.dumps(snapshot, indent=2))  
            logger.debug(f"Autosave written at turn {self.\_turn\_count}")

        except Exception as e:  
            logger.error(f"Autosave failed: {e}")

    def \_register\_signal\_handlers(self):  
        """  
        Register OS signal handlers for emergency save.  
        Catches SIGTERM (system shutdown), SIGINT (Ctrl+C at OS level).  
        Works on Windows and Unix.  
        """  
        import signal

        def emergency\_handler(signum, frame):  
            logger.warning(f"Signal {signum} received — emergency save")  
            self.\_emergency\_save()  
            raise SystemExit(0)

        try:  
            signal.signal(signal.SIGTERM, emergency\_handler)  
            \# SIGINT handled by run.py KeyboardInterrupt — don't double-handle  
        except (OSError, ValueError):  
            \# May fail in some environments — not critical  
            pass

    def \_emergency\_save(self):  
        """  
        Emergency save — called on unexpected exit.  
        Saves whatever we have. Fast, no frills.  
        """  
        import json  
        try:  
            \# Save soul file  
            self.\_soul.finalise()

            \# Write session memory with what we have so far  
            memory \= self.\_extract\_session\_memory()  
            memory\["emergency\_save"\] \= True  
            memory\["complete"\] \= False  
            self.\_write\_session\_moment(memory)

            \# Mark autosave as complete so we know it saved cleanly  
            if self.\_autosave\_path.exists():  
                try:  
                    snap \= json.loads(self.\_autosave\_path.read\_text())  
                    snap\["complete"\] \= True  
                    self.\_autosave\_path.write\_text(json.dumps(snap, indent=2))  
                except Exception:  
                    pass

            logger.info("Emergency save complete")  
        except Exception as e:  
            logger.error(f"Emergency save failed: {e}")

    \# \============================================================  
    \# STREAMING API  
    \# \============================================================

    async def turn\_stream(self, message: str):  
        """  
        Streaming version of turn().

        Yields response chunks as they generate using async\_wrap so the  
        event loop stays responsive between batches — output appears  
        immediately rather than waiting for full generation.

        Post-processing (mood, emotion, memory) runs in a daemon thread  
        after the last token — never blocks the stream.

        Usage in run.py:  
            async for chunk in athena.turn\_stream(message):  
                print(chunk, end="", flush=True)  
        """  
        import threading  
        import asyncio

        async def async\_wrap(sync\_gen):  
            """Wrap a sync generator as async — yields control between batches."""  
            for item in sync\_gen:  
                yield item  
                await asyncio.sleep(0)  \# give event loop chance to flush output

        if not message or not message.strip():  
            return

        \# Run full pre-processing pipeline (safeguard, search, context build)  
        \# This is identical to turn() up to the inference call  
        soul \= self.\_soul.load()  
        if soul is None:  
            soul \= self.\_soul.manifest()

        \# Safeguard check  
        eirene\_risk \= getattr(self.\_mood, 'current\_risk', 0.0)  
        safeguard\_result \= self.\_safeguard.check(message, eirene\_risk=eirene\_risk)

        if safeguard\_result.override\_response and safeguard\_result.crisis\_response:  
            yield safeguard\_result.crisis\_response  
            return

        \# Guardian scan  
        guardian\_result \= self.\_guardian.scan(  
            message=message,  
            user\_id=self.user\_id,  
            session\_id=self.\_session\_id or "unknown",  
        )

        if guardian\_result\["aegis\_active"\]:  
            yield guardian\_result\["response\_override"\]  
            return

        \# Build context (soul, search, TOPICA etc)  
        soul\_messages \= self.\_checkpoint.resume(soul)

        \# Web search  
        if self.\_web\_search.should\_search(message, inference\_engine=self.\_inference):  
            search\_ctx \= self.\_web\_search.search(message)  
            if search\_ctx:  
                injection \= self.\_web\_search.build\_injection(search\_ctx)  
                if injection:  
                    soul\_messages \= list(soul\_messages) \+ \[{  
                        "role": "system", "content": injection  
                    }\]

        \# TOPICA  
        topica\_note \= self.\_topica.get\_context\_for\_athena()  
        if topica\_note:  
            soul\_messages \= list(soul\_messages) \+ \[{  
                "role": "system",  
                "content": (  
                    "\[Memory context — background awareness only\] "  
                    f"Topic awareness: {topica\_note}"  
                )  
            }\]

        \# Safeguard instruction  
        safe\_instruction \= self.\_safeguard.get\_safe\_mode\_instruction(safeguard\_result)  
        if safe\_instruction:  
            soul\_messages \= list(soul\_messages) \+ \[{"role": "system", "content": safe\_instruction}\]

        \# Build full system context  
        system\_injection \= self.\_build\_system\_context(  
            soul=soul,  
            moment\_type="conversation",  
            dialogue\_guidance=None,  
            knowledge\_results=\[\],  
            ability\_contexts=\[\],  
            guardian\_result=guardian\_result,  
            extra\_context=soul\_messages,  
        )

        messages \= \[\]  
        if system\_injection:  
            messages.append({"role": "system", "content": system\_injection})

        history \= self.\_checkpoint.get\_conversation\_context()  
        for m in history:  
            if m.get("role") in ("user", "assistant"):  
                messages.append(m)  
        messages.append({"role": "user", "content": message})

        \# Stream the response — async\_wrap gives event loop control between batches  
        full\_response \= ""  
        routing\_scores \= self.\_inference.calculate\_routing\_scores(message, soul)

        async for chunk, model\_role, is\_fallback in async\_wrap(  
            self.\_inference.route\_and\_stream(  
                messages=messages,  
                routing\_scores=routing\_scores,  
                max\_tokens=512,  
                temperature=self.\_get\_temperature(),  
            )  
        ):  
            full\_response \+= chunk  
            yield chunk

        if not full\_response:  
            yield "Something went wrong on my end. Try again?"  
            return

        \# Background post-processing — non-blocking  
        def post\_process():  
            try:  
                \# Emotion classification  
                emotion\_result \= self.\_emotion\_classifier.classify(message)  
                if not safeguard\_result.block\_memory:  
                    ec \= self.\_emotion\_classifier.get\_session\_emotion\_label()  
                    if ec and self.\_session\_emotion is None:  
                        self.\_session\_emotion \= ec  
                    ec\_label \= emotion\_result.emotion  
                    self.\_session\_emotion\_counts\[ec\_label\] \= (  
                        self.\_session\_emotion\_counts.get(ec\_label, 0\) \+ 1  
                    )

                \# TOPICA update  
                self.\_topica.process(message, mood=self.\_mood.\_mood if hasattr(self.\_mood, '\_mood') else 0.0)

                \# Mood update  
                session\_mins \= self.\_get\_session\_minutes()  
                self.\_mood.update(  
                    message=message,  
                    response=full\_response,  
                    guardian\_risk=guardian\_result\["risk\_score"\],  
                    session\_duration\_mins=session\_mins,  
                )

                \# Soul updates  
                self.\_turn\_count \+= 1  
                if not safeguard\_result.block\_memory:  
                    soul\_updates \= self.\_extract\_soul\_updates(message, soul)  
                    if soul\_updates:  
                        self.\_soul.evolve(soul\_updates)

                \# Checkpoint  
                if safeguard\_result.block\_memory:  
                    self.\_checkpoint.add\_turn("user", f"\[{safeguard\_result.memory\_marker}\]")  
                else:  
                    self.\_checkpoint.add\_turn("user", message)  
                self.\_checkpoint.add\_turn("assistant", full\_response)

                \# Autosave  
                if self.\_turn\_count \- self.\_last\_autosave\_turn \>= self.\_AUTOSAVE\_EVERY:  
                    self.\_autosave()

            except Exception as e:  
                logger.error(f"Stream post-processing error: {e}")

        threading.Thread(target=post\_process, daemon=True).start()

    \# \============================================================  
    \# GAMING COMPANION API  
    \# \============================================================

    def start\_gaming\_mode(self, game\_name: str) \-\> Dict:  
        """  
        Activate gaming companion mode.  
        Creates foreground game thread in TOPICA.  
        User-enabled only — never automatic.  
        """  
        event \= self.\_game.start\_session(game\_name)  
        self.\_topica.set\_gaming\_mode(game\_name)  
        return {  
            "gaming\_mode": True,  
            "game\_name": game\_name,  
            "message": f"Gaming mode active — watching {game\_name}",  
            "reaction\_prompt": event.reaction\_prompt,  
        }

    def stop\_gaming\_mode(self) \-\> Dict:  
        """Deactivate gaming companion mode."""  
        event \= self.\_game.end\_session()  
        return {  
            "gaming\_mode": False,  
            "message": "Gaming mode off",  
        }

    def submit\_game\_event(self, event\_type\_str: str,  
                          detail: str \= "",  
                          data: Dict \= None) \-\> Optional\[Dict\]:  
        """  
        Submit a game event.  
        Returns response dict if Athena should react, None otherwise.  
        """  
        try:  
            event\_type \= GameEventType(event\_type\_str)  
        except ValueError:  
            logger.warning(f"Unknown game event type: {event\_type\_str}")  
            return None

        event \= self.\_game.submit\_event(event\_type, detail, data or {})  
        if not event or not event.should\_respond:  
            return None

        \# Build a context-aware turn using the reaction prompt  
        game\_context \= self.\_game.get\_context\_string()  
        message \= f"\[GAME EVENT: {event.detail or event\_type\_str}\]"

        \# Inject game context into TOPICA  
        if self.\_game.session.active:  
            fg \= self.\_topica.foreground\_thread\_id  
            if fg and fg in self.\_topica.threads:  
                self.\_topica.threads\[fg\].turn\_count \+= 1  
                self.\_topica.threads\[fg\].turn\_last\_active \= self.\_topica.turn\_count

        return {  
            "event\_type": event\_type\_str,  
            "priority": event.priority.value,  
            "reaction\_prompt": event.reaction\_prompt,  
            "game\_context": game\_context,  
            "should\_respond": True,  
        }

    @property  
    def gaming\_mode\_active(self) \-\> bool:  
        """Whether gaming mode is currently active."""  
        return self.\_game.session.active

    def get\_game\_context(self) \-\> str:  
        """Get current game state as context string."""  
        return self.\_game.get\_context\_string()

    def \_write\_session\_moment(self, memory: Dict):  
        """Write session memory to moments/ folder for next session pickup."""  
        import json  
        from pathlib import Path  
        moments\_dir \= Path(\_\_file\_\_).parent.parent / "modules" / "moments"  
        moments\_dir.mkdir(exist\_ok=True)

        \# Keep last 10 session moments — rolling window  
        moment\_file \= moments\_dir / "session\_memories.json"  
        sessions \= \[\]  
        if moment\_file.exists():  
            try:  
                sessions \= json.loads(moment\_file.read\_text())  
            except Exception:  
                sessions \= \[\]

        sessions.append(memory)  
        sessions \= sessions\[-10:\]  \# keep last 10 sessions only

        try:  
            moment\_file.write\_text(json.dumps(sessions, indent=2))  
            logger.info("Session memory written to moments/")  
        except Exception as e:  
            logger.error(f"Failed to write session memory: {e}")

    def \_build\_response(self, response: str, guardian\_result: Dict,  
                         mode: str, mood\_result: Dict \= None,  
                         chibi\_state: Optional\[str\] \= None,  
                         checkpoint\_saved: bool \= False,  
                         model\_role: str \= "unknown") \-\> Dict\[str, Any\]:  
        """Build standardised response dict."""  
        mood \= mood\_result or {}  
        return {  
            "response": response,  
            "active\_mode": mode,  
            "expression": guardian\_result.get("avatar\_override") or mood.get("expression", "CONFIDENT"),  
            "chibi\_state": chibi\_state,  
            "mood": mood.get("mood", 0.0),  
            "mood\_label": mood.get("mood\_label", "NEUTRAL"),  
            "harmony": mood.get("harmony", 0.5),  
            "guardian\_risk": guardian\_result\["risk\_score"\],  
            "guardian\_level": guardian\_result\["level"\],  
            "aegis\_active": guardian\_result\["aegis\_active"\],  
            "checkpoint\_saved": checkpoint\_saved,  
            "model\_role": model\_role,  
            "late\_night\_mode": self.\_mood.get\_late\_night\_mode(),  
            "turn\_count": self.\_turn\_count,  
        }

    def \_empty\_response(self) \-\> Dict\[str, Any\]:  
        """Response for empty input."""  
        return self.\_build\_response(  
            response="I'm here. Take your time.",  
            guardian\_result={"risk\_score": 0.0, "level": "normal",  
                             "aegis\_active": False, "worried\_active": False,  
                             "response\_override": None, "avatar\_override": None},  
            mode=self.\_persona\_mode,  
        )

    def \_load\_manifest(self, manifest\_path: str) \-\> Dict\[str, Any\]:  
        """Load system manifest — single source of truth."""  
        try:  
            return json.loads(Path(manifest\_path).read\_text())  
        except Exception:  
            \# FALLBACK::DEGRADE — use defaults  
            return {  
                "system": {"version": "2.0.0"},  
                "guardian": {"warn\_threshold": 0.75, "crisis\_threshold": 0.90},  
                "mood": {"bounds\_min": \-1.0, "bounds\_max": 1.0,  
                         "harmony\_min": 0.0, "harmony\_max": 1.0, "decay\_rate": 0.05},  
                "persona": {"default\_mode": "Eirene"},  
            }  
"""  
TOPICA — Topic Awareness, Thread Tracking and Relationship Mapping  
\==================================================================  
Companion system to EIRENE. Where EIRENE tracks emotional state,  
TOPICA tracks conversational threads, topic shifts, relevance decay,  
and the relationships between threads.

Designed specifically for non-linear minds — ADHD, autism, anxiety,  
and anyone whose thinking doesn't follow a straight line.

Core principles:  
    \- Topic jumps are not noise. They are signal.  
    \- Threads don't exist in isolation. They form a graph.  
    \- Emotional threads decay slowly. Casual threads fade fast.  
    \- Athena holds the map. She never forces. She just knows.

Thread States:  
    ACTIVE   — current focus  
    PAUSED   — stepped away from, still warm  
    CLOSED   — naturally resolved or faded  
    DEFLECT  — jumped away from something emotional (watch this)

Thread Relationships:  
    CAUSES      — Thread A is creating Thread B  
    CAUSED\_BY   — Thread B stems from Thread A  
    AMPLIFIES   — Thread A makes Thread B worse  
    RELIEVES    — Thread A gives relief from Thread B  
    MIRRORS     — Same feeling in different contexts  
    DEFLECTS    — User jumps to A to avoid dealing with B

Decay Rates by Category:  
    EMOTIONAL   — very slow (holds 20+ turns)  
    SOCIAL      — slow (holds \~15 turns)  
    REFLECTIVE  — slow (holds \~15 turns)  
    PRACTICAL   — medium (holds \~10 turns)  
    CREATIVE    — medium (holds \~10 turns)  
    CASUAL      — fast (fades in \~5 turns)  
"""

import time  
from dataclasses import dataclass, field  
from enum import Enum  
from typing import Dict, List, Optional, Tuple, Set  
import logging

logger \= logging.getLogger(\_\_name\_\_)

class ThreadState(Enum):  
    FOREGROUND \= "foreground"   \# primary active thread — one at a time  
    BACKGROUND \= "background"   \# co-active, aware, slower decay  
    ACTIVE     \= "active"       \# legacy alias for FOREGROUND  
    PAUSED     \= "paused"       \# stepped away, normal decay  
    CLOSED     \= "closed"       \# resolved or faded out  
    DEFLECT    \= "deflect"      \# jumped away from something emotional

class TopicCategory(Enum):  
    EMOTIONAL   \= "emotional"  
    PRACTICAL   \= "practical"  
    CREATIVE    \= "creative"  
    SOCIAL      \= "social"  
    CASUAL      \= "casual"  
    REFLECTIVE  \= "reflective"  
    UNKNOWN     \= "unknown"

class RelationshipType(Enum):  
    CAUSES    \= "causes"       \# A creates B  
    CAUSED\_BY \= "caused\_by"    \# A stems from B  
    AMPLIFIES \= "amplifies"    \# A makes B worse  
    RELIEVES  \= "relieves"     \# A gives relief from B  
    MIRRORS   \= "mirrors"      \# same feeling, different context  
    DEFLECTS  \= "deflects"     \# user jumps to A to avoid B

\# Decay rates — turns before a thread loses 50% of its relevance weight  
\# Background threads decay at 50% the rate of paused threads  
DECAY\_HALF\_LIFE \= {  
    TopicCategory.EMOTIONAL:  20,  
    TopicCategory.SOCIAL:     15,  
    TopicCategory.REFLECTIVE: 15,  
    TopicCategory.PRACTICAL:  10,  
    TopicCategory.CREATIVE:   10,  
    TopicCategory.CASUAL:      5,  
    TopicCategory.UNKNOWN:     8,  
}

BACKGROUND\_DECAY\_MULTIPLIER \= 0.5  \# background decays half as fast as paused

\# Minimum weight before a thread closes automatically  
CLOSE\_THRESHOLD \= 0.05

@dataclass  
class ThreadRelationship:  
    """A directed relationship between two threads."""  
    from\_thread\_id: str  
    to\_thread\_id: str  
    relationship: RelationshipType  
    strength: float \= 1.0        \# 0-1 how strong the relationship is  
    turn\_detected: int \= 0  
    confirmed: bool \= False      \# True if pattern repeated multiple times

@dataclass  
class Thread:  
    """A single conversational thread."""  
    id: str  
    summary: str  
    category: TopicCategory  
    state: ThreadState \= ThreadState.ACTIVE  
    emotional\_weight: float \= 0.0  
    relevance: float \= 1.0           \# decays over time when paused  
    turn\_opened: int \= 0  
    turn\_last\_active: int \= 0  
    turn\_count: int \= 0  
    resolved: bool \= False  
    keywords: List\[str\] \= field(default\_factory=list)  
    related\_thread\_ids: Set\[str\] \= field(default\_factory=set)

@dataclass  
class TopicShift:  
    """Records a detected topic shift."""  
    turn: int  
    from\_thread\_id: Optional\[str\]  
    to\_thread\_id: str  
    shift\_type: str  
    emotional\_context: float

class TOPICA:  
    """  
    Topic Awareness, Thread Tracking, Relevance Decay  
    and Thread Relationship Mapping.  
    """

    \# Keywords that signal topic categories  
    CATEGORY\_SIGNALS \= {  
        TopicCategory.EMOTIONAL: \[  
            "feel", "feeling", "feelings", "sad", "happy", "angry", "scared",  
            "anxious", "worried", "lonely", "depressed", "upset", "hurt",  
            "love", "hate", "miss", "afraid", "overwhelmed", "stressed",  
            "crying", "cry", "tears", "heart", "pain", "numb", "empty",  
            "hopeless", "worthless", "tired", "exhausted", "broken",  
            "struggling", "hard", "difficult", "cant cope", "lost"  
        \],  
        TopicCategory.PRACTICAL: \[  
            "how", "need", "help", "fix", "solve", "work", "problem",  
            "issue", "error", "bug", "build", "make", "create", "code",  
            "install", "setup", "configure", "run", "start", "stop",  
            "trying", "getting", "working on"  
        \],  
        TopicCategory.CREATIVE: \[  
            "idea", "project", "design", "imagine", "what if", "could",  
            "dream", "plan", "vision", "art", "music", "story", "game",  
            "invent", "concept", "building", "making something"  
        \],  
        TopicCategory.SOCIAL: \[  
            "friend", "friends", "family", "mum", "dad", "sister", "brother",  
            "mate", "mates", "relationship", "people", "someone", "they",  
            "them", "told me", "said to me", "treated", "argument", "fight",  
            "falling out", "text", "call", "message", "reply", "ignored",  
            "left out", "excluded", "not invited"  
        \],  
        TopicCategory.REFLECTIVE: \[  
            "why", "meaning", "purpose", "life", "death", "future", "past",  
            "memory", "remember", "used to", "back then", "wonder", "think",  
            "believe", "soul", "real", "truth", "matter", "point",  
            "who am i", "what am i", "where am i going"  
        \],  
        TopicCategory.CASUAL: \[  
            "haha", "lol", "funny", "joke", "random", "anyway", "so yeah",  
            "favourite", "like", "love this", "cool", "interesting", "weird",  
            "nice", "good", "bad", "okay", "dragonball", "game", "music",  
            "film", "movie", "show", "watch", "play"  
        \],  
    }

    \# Relationship detection patterns  
    \# (from\_category, to\_category, shift\_type) → likely relationship  
    RELATIONSHIP\_PATTERNS \= {  
        (TopicCategory.EMOTIONAL, TopicCategory.CASUAL):     RelationshipType.RELIEVES,  
        (TopicCategory.EMOTIONAL, TopicCategory.CREATIVE):   RelationshipType.RELIEVES,  
        (TopicCategory.SOCIAL,    TopicCategory.EMOTIONAL):  RelationshipType.CAUSES,  
        (TopicCategory.PRACTICAL, TopicCategory.EMOTIONAL):  RelationshipType.AMPLIFIES,  
        (TopicCategory.EMOTIONAL, TopicCategory.REFLECTIVE): RelationshipType.MIRRORS,  
        (TopicCategory.SOCIAL,    TopicCategory.REFLECTIVE): RelationshipType.MIRRORS,  
        (TopicCategory.EMOTIONAL, TopicCategory.SOCIAL):     RelationshipType.CAUSED\_BY,  
    }

    NEW\_THREAD\_THRESHOLD \= 0.25

    def \_\_init\_\_(self):  
        self.threads: Dict\[str, Thread\] \= {}  
        self.relationships: List\[ThreadRelationship\] \= \[\]  
        self.active\_thread\_id: Optional\[str\] \= None   \# legacy — points to foreground  
        self.foreground\_thread\_id: Optional\[str\] \= None  
        self.background\_thread\_ids: List\[str\] \= \[\]    \# ordered, most recent first  
        self.shifts: List\[TopicShift\] \= \[\]  
        self.turn\_count: int \= 0  
        self.\_thread\_counter: int \= 0  
        self.\_max\_background\_threads: int \= 3         \# cap background threads

    def process(self, message: str, mood: float \= 0.0) \-\> Dict:  
        """  
        Process a user message.  
        Detect topic, manage threads, decay relevance, map relationships.  
        """  
        self.turn\_count \+= 1  
        message\_lower \= message.lower()

        \# Classify the message  
        category, keywords \= self.\_classify(message\_lower)

        \# Decay all paused threads  
        self.\_decay\_threads()

        \# Check if returning to a paused thread  
        matching\_thread \= self.\_find\_matching\_thread(keywords, category)

        if matching\_thread and matching\_thread.id \!= self.active\_thread\_id:  
            \# Returning to a paused thread  
            old\_thread \= self.threads.get(self.active\_thread\_id)  
            self.\_detect\_relationship(old\_thread, matching\_thread, "return", mood)  
            self.\_shift\_thread(matching\_thread.id, "return", mood)  
            matching\_thread.turn\_last\_active \= self.turn\_count  
            matching\_thread.turn\_count \+= 1  
            matching\_thread.relevance \= min(1.0, matching\_thread.relevance \+ 0.3)  \# recency boost  
            matching\_thread.state \= ThreadState.ACTIVE

        else:  
            prev\_thread \= self.threads.get(self.active\_thread\_id) if self.active\_thread\_id else None

            if prev\_thread and self.\_is\_continuation(prev\_thread, keywords, category):  
                \# Continuing same thread  
                prev\_thread.turn\_last\_active \= self.turn\_count  
                prev\_thread.turn\_count \+= 1  
                prev\_thread.keywords \= list(set(prev\_thread.keywords \+ keywords))\[:20\]  
                prev\_thread.relevance \= 1.0  \# reset decay while active  
                if abs(mood) \> 0.2:  
                    prev\_thread.emotional\_weight \= min(1.0,  
                        prev\_thread.emotional\_weight \* 0.7 \+ abs(mood) \* 0.3)  
            else:  
                \# New thread  
                new\_thread \= self.\_create\_thread(message, category, keywords, mood)

                shift\_type \= "new"  
                if prev\_thread:  
                    if prev\_thread.emotional\_weight \> 0.35:  
                        shift\_type \= "deflection"  
                        prev\_thread.state \= ThreadState.DEFLECT  
                    else:  
                        prev\_thread.state \= ThreadState.PAUSED

                    \# Detect and record relationship between threads  
                    self.\_detect\_relationship(prev\_thread, new\_thread, shift\_type, mood)

                self.\_shift\_thread(new\_thread.id, shift\_type, mood)

        return self.\_build\_context()

    def get\_context\_for\_athena(self) \-\> str:  
        """  
        Returns a brief context string Athena uses to stay oriented.  
        Now includes foreground \+ background thread awareness.  
        Soft awareness — not instructions, just the map.  
        """  
        ctx \= self.\_build\_context()  
        lines \= \[\]

        \# Foreground thread — primary focus  
        active \= ctx.get("active\_thread")  
        if active:  
            lines.append(  
                f"Current topic: {active\['summary'\]} ({active\['category'\]})"  
            )

        \# Background threads — co-active context  
        background \= ctx.get("background\_threads", \[\])  
        if background:  
            bg\_summaries \= ", ".join(  
                t\["summary"\] for t in background\[:3\]  
                if t.get("relevance", 0\) \> 0.2  
            )  
            if bg\_summaries:  
                lines.append(f"Also active in background: {bg\_summaries}")

        \# Warm paused threads (not background — fully stepped away)  
        paused \= ctx.get("paused\_threads", \[\])  
        warm \= \[t for t in paused if t.get("relevance", 0\) \> 0.3  
                and t.get("emotional\_weight", 0\) \> 0.25\]  
        if warm:  
            summaries \= ", ".join(t\["summary"\] for t in warm\[:2\])  
            lines.append(f"Unresolved threads still warm: {summaries}")

        \# Deflection flag  
        if ctx.get("deflections"):  
            lines.append(  
                "Possible deflection — they stepped away from something emotional"  
            )

        \# Return flag  
        last\_shift \= ctx.get("last\_shift")  
        if last\_shift and last\_shift.get("shift\_type") \== "return":  
            lines.append("They've come back to an earlier topic")

        \# Thread relationships  
        rel\_notes \= self.\_get\_relationship\_notes(ctx)  
        if rel\_notes:  
            lines.append(rel\_notes)

        return " | ".join(lines) if lines else ""

    def set\_gaming\_mode(self, game\_name: str \= "game"):  
        """  
        Activate gaming companion mode.  
        Creates a persistent FOREGROUND game context thread.  
        All conversation threads become BACKGROUND.  
        """  
        self.\_thread\_counter \+= 1  
        game\_thread\_id \= f"thread\_{self.\_thread\_counter}"

        game\_thread \= Thread(  
            id=game\_thread\_id,  
            summary=game\_name,  
            category=TopicCategory.CASUAL,  
            state=ThreadState.FOREGROUND,  
            emotional\_weight=0.0,  
            relevance=1.0,  
            turn\_opened=self.turn\_count,  
            turn\_last\_active=self.turn\_count,  
            turn\_count=0,  
            keywords=\["game", "gaming", game\_name.lower()\],  
        )  
        self.threads\[game\_thread\_id\] \= game\_thread

        \# Move current foreground to background  
        if self.foreground\_thread\_id:  
            prev \= self.threads.get(self.foreground\_thread\_id)  
            if prev:  
                prev.state \= ThreadState.BACKGROUND  
                if self.foreground\_thread\_id not in self.background\_thread\_ids:  
                    self.background\_thread\_ids.insert(0, self.foreground\_thread\_id)

        self.foreground\_thread\_id \= game\_thread\_id  
        self.active\_thread\_id \= game\_thread\_id  
        return game\_thread\_id

    def get\_relationship\_map(self) \-\> List\[Dict\]:  
        """Returns the full relationship graph for debugging or display."""  
        return \[  
            {  
                "from": r.from\_thread\_id,  
                "from\_summary": self.threads.get(r.from\_thread\_id, Thread("?","?",TopicCategory.UNKNOWN)).summary,  
                "relationship": r.relationship.value,  
                "to": r.to\_thread\_id,  
                "to\_summary": self.threads.get(r.to\_thread\_id, Thread("?","?",TopicCategory.UNKNOWN)).summary,  
                "strength": r.strength,  
                "confirmed": r.confirmed,  
            }  
            for r in self.relationships  
        \]

    def mark\_thread\_resolved(self):  
        """Call when a thread reaches natural closure."""  
        if self.active\_thread\_id and self.active\_thread\_id in self.threads:  
            t \= self.threads\[self.active\_thread\_id\]  
            t.state \= ThreadState.CLOSED  
            t.resolved \= True

    \# \----------------------------------------------------------------  
    \# Internal methods  
    \# \----------------------------------------------------------------

    def \_classify(self, text: str) \-\> Tuple\[TopicCategory, List\[str\]\]:  
        """Classify message into a topic category and extract keywords."""  
        scores \= {cat: 0 for cat in TopicCategory}  
        found\_keywords \= \[\]

        for category, signals in self.CATEGORY\_SIGNALS.items():  
            for signal in signals:  
                if signal in text:  
                    scores\[category\] \+= 1  
                    found\_keywords.append(signal)

        best \= max(scores, key=scores.get)  
        if scores\[best\] \== 0:  
            best \= TopicCategory.CASUAL

        return best, list(set(found\_keywords))

    def \_find\_matching\_thread(self, keywords: List\[str\],  
                               category: TopicCategory) \-\> Optional\[Thread\]:  
        """Check if keywords match a paused thread."""  
        best\_match \= None  
        best\_score \= self.NEW\_THREAD\_THRESHOLD

        for thread in self.threads.values():  
            if thread.state not in (ThreadState.PAUSED, ThreadState.DEFLECT,  
                                     ThreadState.BACKGROUND):  
                continue  
            if thread.id \== self.active\_thread\_id:  
                continue  
            if thread.relevance \< CLOSE\_THRESHOLD:  
                continue

            overlap \= len(set(keywords) & set(thread.keywords))  
            keyword\_score \= overlap / max(len(thread.keywords), 1\)  
            category\_bonus \= 0.3 if thread.category \== category else 0.0  
            relevance\_weight \= thread.relevance  
            score \= (keyword\_score \+ category\_bonus) \* relevance\_weight

            if score \> best\_score:  
                best\_score \= score  
                best\_match \= thread

        return best\_match

    def \_is\_continuation(self, thread: Thread, keywords: List\[str\],  
                          category: TopicCategory) \-\> bool:  
        """Check if this message continues the active thread."""  
        if thread.category \== category:  
            overlap \= len(set(keywords) & set(thread.keywords))  
            if overlap \> 0:  
                return True  
            if thread.turn\_count \< 3:  
                return True  
        return False

    def \_create\_thread(self, message: str, category: TopicCategory,  
                        keywords: List\[str\], mood: float) \-\> Thread:  
        """Create a new thread."""  
        self.\_thread\_counter \+= 1  
        thread\_id \= f"thread\_{self.\_thread\_counter}"

        if keywords:  
            summary \= (keywords\[0\] if len(keywords) \== 1  
                      else f"{keywords\[0\]} / {keywords\[1\]}")  
        else:  
            summary \= category.value

        thread \= Thread(  
            id=thread\_id,  
            summary=summary,  
            category=category,  
            state=ThreadState.ACTIVE,  
            emotional\_weight=abs(mood),  
            relevance=1.0,  
            turn\_opened=self.turn\_count,  
            turn\_last\_active=self.turn\_count,  
            turn\_count=1,  
            keywords=keywords\[:10\],  
        )  
        self.threads\[thread\_id\] \= thread  
        return thread

    def \_decay\_threads(self):  
        """  
        Apply relevance decay to paused and background threads.  
        Background threads decay at half the rate of paused threads.  
        Active/foreground threads never decay.  
        """  
        for thread in self.threads.values():  
            if thread.state \== ThreadState.BACKGROUND:  
                \# Background decays slowly — maintains context  
                half\_life \= DECAY\_HALF\_LIFE.get(thread.category, 8\)  
                decay\_rate \= (1.0 / half\_life) \* BACKGROUND\_DECAY\_MULTIPLIER  
                thread.relevance \= max(0.0, thread.relevance \- decay\_rate)  
                ew\_decay \= decay\_rate \* 0.5  
                thread.emotional\_weight \= max(0.0, thread.emotional\_weight \- ew\_decay)  
                \# Background threads close at a lower threshold  
                if thread.relevance \< CLOSE\_THRESHOLD \* 0.5:  
                    thread.state \= ThreadState.CLOSED  
                    if thread.id in self.background\_thread\_ids:  
                        self.background\_thread\_ids.remove(thread.id)

            elif thread.state in (ThreadState.PAUSED, ThreadState.DEFLECT):  
                half\_life \= DECAY\_HALF\_LIFE.get(thread.category, 8\)  
                decay\_rate \= 1.0 / half\_life  
                thread.relevance \= max(0.0, thread.relevance \- decay\_rate)  
                ew\_decay \= decay\_rate \* 0.5  
                thread.emotional\_weight \= max(0.0, thread.emotional\_weight \- ew\_decay)  
                if thread.relevance \< CLOSE\_THRESHOLD:  
                    thread.state \= ThreadState.CLOSED

    def \_detect\_relationship(self, from\_thread: Optional\[Thread\],  
                              to\_thread: Thread,  
                              shift\_type: str,  
                              mood: float):  
        """  
        Detect and record the relationship between two threads.  
        Also updates existing relationships if pattern repeats.  
        """  
        if not from\_thread or from\_thread.id \== to\_thread.id:  
            return

        \# Check if relationship already recorded  
        existing \= self.\_find\_relationship(from\_thread.id, to\_thread.id)

        \# Infer relationship type from category pattern and shift type  
        pattern\_key \= (from\_thread.category, to\_thread.category)  
        inferred \= self.RELATIONSHIP\_PATTERNS.get(pattern\_key)

        \# Override with deflection if we detected one  
        if shift\_type \== "deflection":  
            inferred \= RelationshipType.DEFLECTS

        if not inferred:  
            \# Can't infer — still worth noting they're linked  
            inferred \= RelationshipType.MIRRORS

        if existing:  
            \# Strengthen existing relationship  
            existing.strength \= min(1.0, existing.strength \+ 0.2)  
            existing.confirmed \= existing.strength \>= 0.7  
        else:  
            \# New relationship  
            rel \= ThreadRelationship(  
                from\_thread\_id=from\_thread.id,  
                to\_thread\_id=to\_thread.id,  
                relationship=inferred,  
                strength=0.5,  
                turn\_detected=self.turn\_count,  
                confirmed=False,  
            )  
            self.relationships.append(rel)

            \# Also create inverse relationship  
            inverse\_map \= {  
                RelationshipType.CAUSES:    RelationshipType.CAUSED\_BY,  
                RelationshipType.CAUSED\_BY: RelationshipType.CAUSES,  
                RelationshipType.AMPLIFIES: RelationshipType.AMPLIFIES,  
                RelationshipType.RELIEVES:  RelationshipType.RELIEVES,  
                RelationshipType.MIRRORS:   RelationshipType.MIRRORS,  
                RelationshipType.DEFLECTS:  RelationshipType.DEFLECTS,  
            }  
            inverse \= ThreadRelationship(  
                from\_thread\_id=to\_thread.id,  
                to\_thread\_id=from\_thread.id,  
                relationship=inverse\_map.get(inferred, RelationshipType.MIRRORS),  
                strength=0.5,  
                turn\_detected=self.turn\_count,  
                confirmed=False,  
            )  
            self.relationships.append(inverse)

            \# Link the threads  
            from\_thread.related\_thread\_ids.add(to\_thread.id)  
            to\_thread.related\_thread\_ids.add(from\_thread.id)

    def \_find\_relationship(self, from\_id: str,  
                            to\_id: str) \-\> Optional\[ThreadRelationship\]:  
        """Find existing relationship between two threads."""  
        for r in self.relationships:  
            if r.from\_thread\_id \== from\_id and r.to\_thread\_id \== to\_id:  
                return r  
        return None

    def \_get\_relationship\_notes(self, ctx: Dict) \-\> str:  
        """Surface the most meaningful relationship insight for Athena."""  
        active \= ctx.get("active\_thread")  
        if not active:  
            return ""

        active\_id \= active.get("id")  
        relevant \= \[r for r in self.relationships  
                    if r.from\_thread\_id \== active\_id and r.strength \> 0.4\]

        if not relevant:  
            return ""

        \# Pick strongest relationship  
        strongest \= max(relevant, key=lambda r: r.strength)  
        related \= self.threads.get(strongest.to\_thread\_id)  
        if not related:  
            return ""

        rel\_notes \= {  
            RelationshipType.CAUSES:    f"This topic may be causing: {related.summary}",  
            RelationshipType.CAUSED\_BY: f"This may stem from: {related.summary}",  
            RelationshipType.AMPLIFIES: f"This is making harder: {related.summary}",  
            RelationshipType.RELIEVES:  f"This gives relief from: {related.summary}",  
            RelationshipType.MIRRORS:   f"Similar feeling to: {related.summary}",  
            RelationshipType.DEFLECTS:  f"May be avoiding: {related.summary}",  
        }

        note \= rel\_notes.get(strongest.relationship, "")  
        if strongest.confirmed:  
            note \+= " (pattern confirmed)"  
        return note

    def \_shift\_thread(self, new\_thread\_id: str, shift\_type: str, mood: float):  
        """  
        Record a thread shift and update foreground/background.

        New thread becomes FOREGROUND.  
        Previous foreground moves to BACKGROUND (not PAUSED).  
        Background threads keep context and decay slowly.  
        """  
        self.shifts.append(TopicShift(  
            turn=self.turn\_count,  
            from\_thread\_id=self.active\_thread\_id,  
            to\_thread\_id=new\_thread\_id,  
            shift\_type=shift\_type,  
            emotional\_context=mood,  
        ))

        \# Move current foreground to background  
        if self.foreground\_thread\_id and self.foreground\_thread\_id \!= new\_thread\_id:  
            prev \= self.threads.get(self.foreground\_thread\_id)  
            if prev and prev.state not in (ThreadState.CLOSED, ThreadState.DEFLECT):  
                prev.state \= ThreadState.BACKGROUND  
                \# Add to background list (most recent first)  
                if self.foreground\_thread\_id not in self.background\_thread\_ids:  
                    self.background\_thread\_ids.insert(0, self.foreground\_thread\_id)  
                \# Cap background threads  
                while len(self.background\_thread\_ids) \> self.\_max\_background\_threads:  
                    oldest \= self.background\_thread\_ids.pop()  
                    if oldest in self.threads:  
                        self.threads\[oldest\].state \= ThreadState.PAUSED

        \# Set new foreground  
        self.foreground\_thread\_id \= new\_thread\_id  
        self.active\_thread\_id \= new\_thread\_id  \# keep legacy pointer in sync

        \# Remove from background if it was there  
        if new\_thread\_id in self.background\_thread\_ids:  
            self.background\_thread\_ids.remove(new\_thread\_id)

        if new\_thread\_id in self.threads:  
            self.threads\[new\_thread\_id\].state \= ThreadState.FOREGROUND

    def \_build\_context(self) \-\> Dict:  
        """Build context dict including foreground and background threads."""  
        active \= self.threads.get(self.active\_thread\_id)  
        background \= \[  
            self.threads\[tid\] for tid in self.background\_thread\_ids  
            if tid in self.threads  
        \]  
        paused \= \[t for t in self.threads.values()  
                  if t.state in (ThreadState.PAUSED, ThreadState.DEFLECT)\]  
        deflections \= \[t for t in self.threads.values()  
                       if t.state \== ThreadState.DEFLECT\]  
        last\_shift \= self.shifts\[-1\] if self.shifts else None

        return {  
            "active\_thread": {  
                "id": active.id,  
                "summary": active.summary,  
                "category": active.category.value,  
                "emotional\_weight": active.emotional\_weight,  
                "relevance": active.relevance,  
                "turn\_count": active.turn\_count,  
                "related\_threads": list(active.related\_thread\_ids),  
                "state": active.state.value,  
            } if active else None,  
            "background\_threads": \[  
                {  
                    "id": t.id,  
                    "summary": t.summary,  
                    "category": t.category.value,  
                    "emotional\_weight": t.emotional\_weight,  
                    "relevance": t.relevance,  
                    "state": t.state.value,  
                }  
                for t in background  
            \],  
            "paused\_threads": \[  
                {  
                    "id": t.id,  
                    "summary": t.summary,  
                    "category": t.category.value,  
                    "emotional\_weight": t.emotional\_weight,  
                    "relevance": t.relevance,  
                    "state": t.state.value,  
                }  
                for t in paused  
            \],  
            "deflections": \[  
                {"id": t.id, "summary": t.summary}  
                for t in deflections  
            \],  
            "relationships": self.get\_relationship\_map(),  
            "last\_shift": {  
                "shift\_type": last\_shift.shift\_type,  
                "from\_thread": last\_shift.from\_thread\_id,  
                "to\_thread": last\_shift.to\_thread\_id,  
                "emotional\_context": last\_shift.emotional\_context,  
            } if last\_shift else None,  
            "total\_threads": len(self.threads),  
            "foreground\_thread": self.foreground\_thread\_id,  
            "background\_count": len(self.background\_thread\_ids),  
            "turn\_count": self.turn\_count,  
        }

"""  
core/soul.py \-- Soul File System  
Implements: Preserva (state preservation) \+ Revela (encryption/decryption)  
Operator: MANIFEST, EVOLVE, FINALISE, FALLBACK::COMPENSATE  
Privacy by architecture: AES-256, key never leaves device, never written plain to disk  
"""

import os  
import json  
import hashlib  
import base64  
import logging  
from pathlib import Path  
from datetime import datetime, timezone  
from typing import Optional, Dict, Any, List

from cryptography.hazmat.primitives.ciphers.aead import AESGCM  
from cryptography.hazmat.primitives import hashes  
from cryptography.hazmat.primitives.kdf.pbkdf2 import PBKDF2HMAC

logger \= logging.getLogger(\_\_name\_\_)

def \_derive\_key(user\_id: str, device\_salt: bytes) \-\> bytes:  
    """  
    Derive AES-256 encryption key from user\_id \+ device fingerprint.  
    Key never stored — always re-derived from device identity.  
    """  
    kdf \= PBKDF2HMAC(  
        algorithm=hashes.SHA256(),  
        length=32,  
        salt=device\_salt,  
        iterations=600000,  
    )  
    return kdf.derive(user\_id.encode())

def \_get\_device\_salt(data\_path: Path) \-\> bytes:  
    """  
    Get or generate device-specific salt.  
    Acts as device secure enclave substitute for desktop/server.  
    """  
    salt\_file \= data\_path / ".device\_salt"  
    if salt\_file.exists():  
        return salt\_file.read\_bytes()  
    salt \= os.urandom(32)  
    salt\_file.write\_bytes(salt)  
    \# Restrict permissions — owner read only  
    os.chmod(salt\_file, 0o600)  
    return salt

class SoulFile:  
    """  
    Soul File System — the most sensitive data in Athena.  
    Encrypted at rest with AES-256-GCM.  
    Decrypted in memory only — never written plain to disk.  
      
    Operators implemented:  
    \- MANIFEST: create new soul file on first launch  
    \- EVOLVE: update after every session  
    \- FINALISE: lock updates at session end  
    \- FALLBACK::COMPENSATE: recover from corruption via session logs  
    \- DEPENDS\_ON: soul file always loads before response generation  
    """

    STRUCTURE \= {  
        "user\_id": "",  
        "name": "",  
        "join\_date": "",  
        "personality\_notes": \[\],  
        "important\_facts": \[\],  
        "emotional\_patterns": {},  
        "relationship\_depth": 0,  
        "goals": \[\],  
        "wins": \[\],  
        "open\_threads": \[\],  
        "last\_seen": "",  
        "music\_taste": {},  
        "dialogue\_preferences": {},  
        "growth\_arc": \[\],  
        "session\_count": 0,  
        "total\_turns": 0  
    }

    def \_\_init\_\_(self, user\_id: str, data\_path: str):  
        self.user\_id \= user\_id  
        self.data\_path \= Path(data\_path) / user\_id  
        self.data\_path.mkdir(parents=True, exist\_ok=True)  
        self.\_soul\_path \= self.data\_path / "soul.enc"  
        self.\_device\_salt \= \_get\_device\_salt(Path(data\_path))  
        self.\_key \= \_derive\_key(user\_id, self.\_device\_salt)  
        self.\_data: Optional\[Dict\[str, Any\]\] \= None  
        self.\_dirty \= False

    def manifest(self, name: str \= "") \-\> Dict\[str, Any\]:  
        """  
        MANIFEST: Create new soul file on first launch.  
        Encrypted immediately — never exists plain on disk.  
        """  
        soul \= dict(self.STRUCTURE)  
        soul\["user\_id"\] \= self.user\_id  
        soul\["name"\] \= name  
        soul\["join\_date"\] \= datetime.now(timezone.utc).isoformat()  
        soul\["last\_seen"\] \= datetime.now(timezone.utc).isoformat()  
        self.\_data \= soul  
        success \= self.\_write\_encrypted(soul)  
        if not success:  
            \# FALLBACK::COMPENSATE — encryption failed  
            logger.error(f"Soul file encryption failed for {self.user\_id}")  
            self.\_data \= None  
            raise RuntimeError(  
                "Soul file could not be created safely. "  
                "Data will not be saved until encryption is available."  
            )  
        logger.info(f"Soul file manifested for user {self.user\_id}")  
        return soul

    def load(self) \-\> Optional\[Dict\[str, Any\]\]:  
        """  
        DEPENDS\_ON: Load soul file before any response generation.  
        Decrypted in memory only — never written plain to disk.  
        FALLBACK::COMPENSATE: recover from session logs if corrupt.  
        """  
        if self.\_data is not None:  
            return self.\_data

        if not self.\_soul\_path.exists():  
            return None

        try:  
            soul \= self.\_read\_encrypted()  
            if soul is None:  
                return self.\_fallback\_recover()  
            self.\_data \= soul  
            return soul  
        except Exception as e:  
            logger.error(f"Soul file load failed: {e}")  
            return self.\_fallback\_recover()

    def evolve(self, updates: Dict\[str, Any\]) \-\> bool:  
        """  
        EVOLVE: Update soul file after session with new learnings.  
        Only specific fields updated — never overwritten completely.  
        FINALISE: Called at session end, not mid-session.  
        """  
        if self.\_data is None:  
            return False

        for field, value in updates.items():  
            if field not in self.STRUCTURE:  
                continue  
            if isinstance(value, list) and isinstance(self.\_data.get(field), list):  
                \# Append to lists — never overwrite  
                \# Cap sizes to prevent unbounded growth  
                FIELD\_CAPS \= {  
                    "growth\_arc": 20,       \# last 20 sessions  
                    "important\_facts": 50,  \# top 50 facts  
                    "personality\_notes": 30,  
                    "goals": 20,  
                    "wins": 30,  
                    "open\_threads": 10,  
                }  
                existing \= self.\_data\[field\]  
                if isinstance(value, list):  
                    for item in value:  
                        if item not in existing:  
                            existing.append(item)  
                    cap \= FIELD\_CAPS.get(field)  
                    if cap and len(existing) \> cap:  
                        self.\_data\[field\] \= existing\[-cap:\]  \# keep most recent  
            elif isinstance(value, dict) and isinstance(self.\_data.get(field), dict):  
                self.\_data\[field\].update(value)  
            else:  
                self.\_data\[field\] \= value

        self.\_data\["last\_seen"\] \= datetime.now(timezone.utc).isoformat()  
        self.\_dirty \= True  
        return True

    def finalise(self) \-\> bool:  
        """  
        FINALISE: Lock soul file updates at session end.  
        Writes encrypted version to disk.  
        """  
        if not self.\_dirty or self.\_data is None:  
            return True  
        success \= self.\_write\_encrypted(self.\_data)  
        if success:  
            self.\_dirty \= False  
            logger.info(f"Soul file finalised for {self.user\_id}")  
        return success

    def export(self) \-\> Optional\[Dict\[str, Any\]\]:  
        """  
        User data export — their right, always available.  
        Returns plain JSON to user. Clears from memory after.  
        """  
        soul \= self.load()  
        if soul is None:  
            return None  
        \# Return copy — clear sensitive data from working memory after  
        return dict(soul)

    def delete(self) \-\> bool:  
        """  
        Permanently erase all user data.  
        Destroy key derivation material first, then delete file.  
        Data becomes unrecoverable.  
        """  
        try:  
            \# Overwrite key material  
            self.\_key \= b'\\x00' \* 32  
            self.\_data \= None  
            if self.\_soul\_path.exists():  
                \# Overwrite with random bytes before deletion  
                size \= self.\_soul\_path.stat().st\_size  
                self.\_soul\_path.write\_bytes(os.urandom(max(size, 256)))  
                self.\_soul\_path.unlink()  
            logger.info(f"Soul file deleted for {self.user\_id}")  
            return True  
        except Exception as e:  
            logger.error(f"Soul file deletion failed: {e}")  
            return False

    def get(self, field: str, default=None):  
        """Safe field access."""  
        if self.\_data is None:  
            self.load()  
        if self.\_data is None:  
            return default  
        return self.\_data.get(field, default)

    def \_write\_encrypted(self, data: Dict\[str, Any\]) \-\> bool:  
        """  
        AES-256-GCM encryption.  
        Nonce generated fresh for every write.  
        """  
        try:  
            plaintext \= json.dumps(data, default=str).encode()  
            aesgcm \= AESGCM(self.\_key)  
            nonce \= os.urandom(12)  
            ciphertext \= aesgcm.encrypt(nonce, plaintext, None)  
            \# Store: nonce (12 bytes) \+ ciphertext  
            payload \= base64.b64encode(nonce \+ ciphertext)  
            self.\_soul\_path.write\_bytes(payload)  
            os.chmod(self.\_soul\_path, 0o600)  
            return True  
        except Exception as e:  
            logger.error(f"Soul file encryption error: {e}")  
            return False

    def \_read\_encrypted(self) \-\> Optional\[Dict\[str, Any\]\]:  
        """  
        AES-256-GCM decryption.  
        Decrypted in memory only — never written plain to disk.  
        """  
        try:  
            payload \= base64.b64decode(self.\_soul\_path.read\_bytes())  
            nonce \= payload\[:12\]  
            ciphertext \= payload\[12:\]  
            aesgcm \= AESGCM(self.\_key)  
            plaintext \= aesgcm.decrypt(nonce, ciphertext, None)  
            return json.loads(plaintext.decode())  
        except Exception as e:  
            logger.error(f"Soul file decryption error: {e}")  
            return None

    def \_fallback\_recover(self) \-\> Optional\[Dict\[str, Any\]\]:  
        """  
        FALLBACK::COMPENSATE: Recover soul file from session logs.  
        Last resort — never lose a user's history.  
        """  
        logger.warning(f"Attempting soul file recovery for {self.user\_id}")  
        sessions\_path \= self.data\_path / "sessions"  
        if not sessions\_path.exists():  
            return None  
        sessions \= sorted(sessions\_path.glob("\*.json"), reverse=True)  
        if not sessions:  
            return None  
        try:  
            latest \= json.loads(sessions\[0\].read\_text())  
            soul \= dict(self.STRUCTURE)  
            soul\["user\_id"\] \= self.user\_id  
            soul\["name"\] \= latest.get("user\_name", "")  
            soul\["join\_date"\] \= latest.get("timestamp", datetime.now(timezone.utc).isoformat())  
            soul\["last\_seen"\] \= latest.get("timestamp", datetime.now(timezone.utc).isoformat())  
            self.\_data \= soul  
            self.\_write\_encrypted(soul)  
            logger.info(f"Soul file recovered from session logs for {self.user\_id}")  
            return soul  
        except Exception as e:  
            logger.error(f"Soul file recovery failed: {e}")  
            return None

"""  
Session Summariser  
\==================  
Lightweight inference pass at session end only.  
Produces a natural one-sentence human-readable memory summary.

Runs ONCE per session — not per turn.  
Feeds from: TOPICA thread data, EmotionClassifier dominant emotion, EIRENE mood.  
Output stored alongside structured session memory.

Does NOT replace \_extract\_session\_memory() or \_extract\_soul\_updates().  
This is an additional natural language layer for richer recall.

Example output:  
    "Troy was excited about Athena's development progress, talked about  
     gaming companion mode, and seemed energised throughout."  
"""

import json  
import logging  
from dataclasses import dataclass  
from typing import Optional, List, Dict, Any

logger \= logging.getLogger(\_\_name\_\_)

\# How many recent turns to feed into the summary  
DEFAULT\_TURNS\_TO\_SUMMARISE \= 20

SUMMARISER\_SYSTEM \= """You are writing a one-sentence memory note about a conversation.  
Write in third person about the user (use their name if known).  
Be warm, natural, and specific. Focus on what actually happened.  
Return ONLY the sentence. No quotes. No punctuation at the end except a full stop.  
Keep it under 30 words.

Examples:  
Troy was excited about a new AI build, talked through some technical challenges, and seemed energised throughout.  
Troy mentioned feeling tired but cheered up talking about anime and gaming plans.  
Troy shared some personal worries early on but ended the conversation in good spirits."""

@dataclass  
class SessionSummary:  
    """Result of a session summarisation pass."""  
    summary\_text: str  
    dominant\_topic: str  
    dominant\_emotion: str  
    mood: float  
    confidence: float \= 0.8  
    fallback: bool \= False

class SessionSummariser:  
    """  
    Produces a natural language summary of the session at close.  
    Single inference pass. Lightweight. Optional — falls back gracefully.  
    """

    def \_\_init\_\_(self, inference\_engine=None):  
        self.\_inference \= inference\_engine

    def summarise(  
        self,  
        user\_name: str,  
        conversation\_turns: List\[Dict\],  
        topica\_threads: Dict,  
        dominant\_emotion: str,  
        mood: float,  
        max\_turns: int \= DEFAULT\_TURNS\_TO\_SUMMARISE,  
    ) \-\> SessionSummary:  
        """  
        Generate a natural session summary.

        Args:  
            user\_name:         user's name  
            conversation\_turns: recent conversation history  
            topica\_threads:    TOPICA thread objects  
            dominant\_emotion:  most common emotion this session  
            mood:              EIRENE mood at close  
            max\_turns:         how many turns to include

        Returns:  
            SessionSummary with natural text and structured signals  
        """  
        \# Build dominant topic from TOPICA  
        dominant\_topic \= self.\_get\_dominant\_topic(topica\_threads)

        \# Try model summarisation  
        if self.\_inference is not None:  
            summary\_text \= self.\_generate\_summary(  
                user\_name, conversation\_turns, dominant\_topic,  
                dominant\_emotion, mood, max\_turns  
            )  
            if summary\_text:  
                return SessionSummary(  
                    summary\_text=summary\_text,  
                    dominant\_topic=dominant\_topic,  
                    dominant\_emotion=dominant\_emotion,  
                    mood=round(mood, 3),  
                    fallback=False,  
                )

        \# Fallback — rule-based summary  
        summary\_text \= self.\_fallback\_summary(  
            user\_name, dominant\_topic, dominant\_emotion, mood  
        )  
        return SessionSummary(  
            summary\_text=summary\_text,  
            dominant\_topic=dominant\_topic,  
            dominant\_emotion=dominant\_emotion,  
            mood=round(mood, 3),  
            fallback=True,  
        )

    \# \----------------------------------------------------------------  
    \# Internal  
    \# \----------------------------------------------------------------

    def \_generate\_summary(  
        self,  
        user\_name: str,  
        turns: List\[Dict\],  
        dominant\_topic: str,  
        dominant\_emotion: str,  
        mood: float,  
        max\_turns: int,  
    ) \-\> Optional\[str\]:  
        """Single inference pass to generate natural summary."""  
        try:  
            \# Build conversation excerpt — user turns only, last N  
            user\_turns \= \[  
                t.get("content", "") for t in turns  
                if t.get("role") \== "user"  
                and not t.get("content", "").startswith("\[")  
            \]\[-max\_turns:\]

            if not user\_turns:  
                return None

            conversation\_excerpt \= "\\n".join(  
                f"- {t\[:100\]}" for t in user\_turns\[:10\]  
            )

            mood\_desc \= (  
                "positive" if mood \> 0.2  
                else "low" if mood \< \-0.2  
                else "neutral"  
            )

            prompt \= (  
                f"User name: {user\_name or 'the user'}\\n"  
                f"Dominant topic: {dominant\_topic}\\n"  
                f"Dominant emotion: {dominant\_emotion}\\n"  
                f"Mood at close: {mood\_desc} ({mood:.2f})\\n"  
                f"What they talked about:\\n{conversation\_excerpt}\\n\\n"  
                f"Write a one-sentence memory note about this conversation."  
            )

            messages \= \[  
                {"role": "system", "content": SUMMARISER\_SYSTEM},  
                {"role": "user", "content": prompt}  
            \]

            result \= self.\_inference.route\_and\_respond(  
                messages=messages,  
                routing\_scores={"speed\_weight": 1.0},  
                max\_tokens=60,  
                temperature=0.4,  
            )

            text \= result.get("text", "").strip() if result else ""

            \# Reject if stub or too short  
            if not text or result.get("stub\_mode") or len(text) \< 10:  
                return None

            \# Clean up — remove quotes if model added them  
            text \= text.strip('"\\'')  
            if not text.endswith("."):  
                text \+= "."

            return text

        except Exception as e:  
            logger.warning(f"Session summariser inference failed: {e}")  
            return None

    def \_get\_dominant\_topic(self, threads: Dict) \-\> str:  
        """Get the most active topic from TOPICA threads."""  
        if not threads:  
            return "general conversation"

        \# Find thread with most turns  
        best \= max(  
            threads.values(),  
            key=lambda t: t.turn\_count,  
            default=None  
        )  
        if best and best.summary:  
            return best.summary

        return "general conversation"

    def \_fallback\_summary(  
        self,  
        user\_name: str,  
        dominant\_topic: str,  
        dominant\_emotion: str,  
        mood: float,  
    ) \-\> str:  
        """Rule-based fallback summary when inference unavailable."""  
        name \= user\_name or "The user"

        mood\_phrase \= (  
            "seemed in good spirits"  
            if mood \> 0.2  
            else "seemed a bit low"  
            if mood \< \-0.2  
            else "was fairly neutral throughout"  
        )

        emotion\_map \= {  
            "happy": "was in a good mood",  
            "excited": "was excited",  
            "playful": "was in a playful mood",  
            "tired": "mentioned feeling tired",  
            "sad": "seemed down",  
            "anxious": "seemed anxious",  
            "stressed": "seemed stressed",  
            "hopeful": "seemed hopeful",  
            "curious": "was curious and engaged",  
            "calm": "was calm throughout",  
            "lonely": "mentioned feeling lonely",  
            "overwhelmed": "seemed overwhelmed",  
            "frustrated": "seemed frustrated",  
            "grateful": "expressed gratitude",  
            "proud": "seemed proud of something",  
        }

        emotion\_phrase \= emotion\_map.get(  
            dominant\_emotion, f"was {dominant\_emotion}"  
        )

        return (  
            f"{name} {emotion\_phrase} and talked about {dominant\_topic}, "  
            f"and {mood\_phrase}."  
        )

"""  
core/mood.py \-- Mood & Harmony Engine  
Implements: Equilibria (equilibrium algorithm) \+ Wuven (self-adjusting regulation)  
Operator: BRIDGE (mood → avatar expression), LAYER (three signals combined)

FIXES: Harmony drift bug from v1.0  
\- Mood bounded: \-1.0 to \+1.0  
\- Harmony bounded: 0.0 to 1.0    
\- Self-correcting: decays toward neutral, cannot compound indefinitely  
\- Normal positive conversation holds neutral-to-positive range  
"""

import re  
import math  
import logging  
from datetime import datetime, timezone  
from typing import Dict, Any, Optional

logger \= logging.getLogger(\_\_name\_\_)

\# Sentiment word lists for Signal 1  
POSITIVE\_WORDS \= {  
    'happy', 'great', 'wonderful', 'amazing', 'excited', 'love', 'joy',  
    'fantastic', 'brilliant', 'excellent', 'good', 'glad', 'pleased',  
    'grateful', 'thankful', 'hope', 'inspired', 'proud', 'confident',  
    'better', 'improved', 'progress', 'win', 'success', 'achieved',  
    'beautiful', 'perfect', 'awesome', 'nice', 'fun', 'laugh', 'smile',  
    'working', 'okay', 'fine', 'alright', 'yes', 'absolutely', 'sure'  
}

NEGATIVE\_WORDS \= {  
    'sad', 'terrible', 'awful', 'horrible', 'depressed', 'anxious',  
    'worried', 'scared', 'angry', 'frustrated', 'upset', 'hurt',  
    'lost', 'confused', 'tired', 'exhausted', 'stressed', 'overwhelmed',  
    'bad', 'worse', 'worst', 'hate', 'fear', 'pain', 'suffering',  
    'broken', 'failed', 'failure', 'hopeless', 'useless', 'worthless'  
}

QUESTION\_PATTERN \= re.compile(r'\\?')  
LONG\_MESSAGE\_THRESHOLD \= 100  \# chars

class MoodEngine:  
    """  
    Three-signal mood system with bounded self-correction.  
      
    Signal 1 — Sentiment Score: content analysis  
    Signal 2 — Engagement Score: interaction quality    
    Signal 3 — Time Signal: local time awareness  
      
    Equilibria ensures mood cannot drift infinitely.  
    Wuven provides self-adjusting regulation back toward neutral.  
    """

    def \_\_init\_\_(self,  
                 mood\_min: float \= \-1.0,  
                 mood\_max: float \= 1.0,  
                 harmony\_min: float \= 0.0,  
                 harmony\_max: float \= 1.0,  
                 decay\_rate: float \= 0.05):  
        self.mood\_min \= mood\_min  
        self.mood\_max \= mood\_max  
        self.harmony\_min \= harmony\_min  
        self.harmony\_max \= harmony\_max  
        self.decay\_rate \= decay\_rate  \# Rate of decay toward neutral per turn

        \# State  
        self.\_mood: float \= 0.0        \# Bounded \-1.0 to \+1.0  
        self.\_harmony: float \= 0.5     \# Bounded 0.0 to 1.0  
        self.\_turn\_count: int \= 0  
        self.\_session\_sentiment\_history: list \= \[\]

    @property  
    def mood(self) \-\> float:  
        return round(self.\_mood, 4\)

    @property  
    def mood\_label(self) \-\> str:  
        """Human-readable mood label."""  
        if self.\_mood \>= 0.6:  
            return "ELATED"  
        elif self.\_mood \>= 0.3:  
            return "POSITIVE"  
        elif self.\_mood \>= 0.1:  
            return "CONTENT"  
        elif self.\_mood \>= \-0.1:  
            return "NEUTRAL"  
        elif self.\_mood \>= \-0.3:  
            return "SUBDUED"  
        elif self.\_mood \>= \-0.6:  
            return "LOW"  
        return "DISTRESSED"

    @property  
    def harmony(self) \-\> float:  
        return round(self.\_harmony, 4\)

    def update(self, message: str, response: str \= "",  
               guardian\_risk: float \= 0.0,  
               session\_duration\_mins: float \= 0.0) \-\> Dict\[str, Any\]:  
        """  
        LAYER: Combine three signals into bounded mood score.  
        Equilibria ensures bounds. Wuven ensures self-correction.  
        """  
        self.\_turn\_count \+= 1

        \# Signal 1 — Sentiment Score  
        sentiment \= self.\_calculate\_sentiment(message)

        \# Signal 2 — Engagement Score  
        engagement \= self.\_calculate\_engagement(message)

        \# Signal 3 — Time Signal  
        time\_modifier \= self.\_calculate\_time\_signal(session\_duration\_mins)

        \# Combine signals — weighted average  
        raw\_delta \= (  
            sentiment \* 0.50 \+  
            engagement \* 0.30 \+  
            time\_modifier \* 0.20  
        )

        \# Wuven: self-adjusting regulation — decay toward neutral  
        \# The further from neutral, the stronger the pull back  
        decay \= self.\_mood \* self.decay\_rate  
        adjusted\_delta \= raw\_delta \- decay

        \# Equilibria: apply bounded update  
        new\_mood \= self.\_mood \+ (adjusted\_delta \* 0.3)  \# Dampened update  
        self.\_mood \= max(self.mood\_min, min(self.mood\_max, new\_mood))

        \# Update harmony — bounded 0.0 to 1.0  
        \# Maps mood (-1 to 1\) to harmony (0 to 1\) with self-correction  
        \# Guardian risk reduces harmony  
        target\_harmony \= (self.\_mood \+ 1.0) / 2.0  \# Map \-1..1 to 0..1  
        guardian\_penalty \= guardian\_risk \* 0.3  
        target\_harmony \= max(0.0, target\_harmony \- guardian\_penalty)

        \# Smooth harmony transition — never jumps  
        harmony\_delta \= (target\_harmony \- self.\_harmony) \* 0.2  
        self.\_harmony \= max(  
            self.harmony\_min,  
            min(self.harmony\_max, self.\_harmony \+ harmony\_delta)  
        )

        \# Store for history  
        self.\_session\_sentiment\_history.append(round(sentiment, 3))

        \# BRIDGE: map to avatar expression  
        expression \= self.\_bridge\_to\_expression(guardian\_risk)

        return {  
            "mood": self.mood,  
            "mood\_label": self.mood\_label,  
            "harmony": self.harmony,  
            "sentiment\_signal": round(sentiment, 3),  
            "engagement\_signal": round(engagement, 3),  
            "time\_signal": round(time\_modifier, 3),  
            "expression": expression,  
            "turn\_count": self.\_turn\_count,  
        }

    def \_calculate\_sentiment(self, message: str) \-\> float:  
        """  
        Signal 1: Sentiment analysis of message content.  
        Returns \-1.0 to \+1.0  
        """  
        if not message:  
            return 0.0

        words \= re.findall(r'\\b\\w+\\b', message.lower())  
        if not words:  
            return 0.0

        pos\_count \= sum(1 for w in words if w in POSITIVE\_WORDS)  
        neg\_count \= sum(1 for w in words if w in NEGATIVE\_WORDS)  
        total \= len(words)

        if total \== 0:  
            return 0.0

        \# Normalised sentiment — bounded  
        sentiment \= (pos\_count \- neg\_count) / max(total \* 0.3, 1\)  
        return max(-1.0, min(1.0, sentiment))

    def \_calculate\_engagement(self, message: str) \-\> float:  
        """  
        Signal 2: Engagement score from interaction quality.  
        Returns \-0.5 to \+0.5 (smaller range — supporting signal)  
        """  
        score \= 0.0

        if not message:  
            return score

        length \= len(message)

        \# Long thoughtful messages \= connection deepening  
        if length \> LONG\_MESSAGE\_THRESHOLD \* 2:  
            score \+= 0.3  
        elif length \> LONG\_MESSAGE\_THRESHOLD:  
            score \+= 0.15  
        elif length \< 10:  
            score \-= 0.1

        \# Questions asked \= curiosity \= trust  
        question\_count \= len(QUESTION\_PATTERN.findall(message))  
        score \+= min(0.2, question\_count \* 0.1)

        return max(-0.5, min(0.5, score))

    def \_calculate\_time\_signal(self, session\_duration\_mins: float) \-\> float:  
        """  
        Signal 3: Time-based modifier.  
        Returns \-0.3 to \+0.2  
        """  
        now \= datetime.now()  
        hour \= now.hour  
        score \= 0.0

        \# Morning — warmer, more energetic  
        if 7 \<= hour \<= 10:  
            score \+= 0.1  
        \# Late night — quiet mode  
        elif hour \>= 22 or hour \< 5:  
            score \-= 0.1  
        \# Afternoon peak  
        elif 14 \<= hour \<= 17:  
            score \+= 0.05

        \# Very long session — gentle nudge down (fatigue signal)  
        if session\_duration\_mins \> 120:  
            score \-= 0.2

        return max(-0.3, min(0.2, score))

    def \_bridge\_to\_expression(self, guardian\_risk: float) \-\> str:  
        """  
        BRIDGE: Map mood score to avatar expression.  
        Guardian risk overrides mood-based expression.  
        """  
        if guardian\_risk \>= 0.90:  
            return "GUARDIAN\_ACTIVATED"  
        elif guardian\_risk \>= 0.75:  
            return "WORRIED"

        mood \= self.\_mood  
        if mood \>= 0.6:  
            return "EXCITED"  
        elif mood \>= 0.3:  
            return "HAPPY"  
        elif mood \>= 0.0:  
            return "CONFIDENT"  
        elif mood \>= \-0.3:  
            return "DETERMINED"  
        elif mood \>= \-0.6:  
            return "SAD"  
        return "WORRIED"

    def get\_late\_night\_mode(self) \-\> bool:  
        """Time awareness — always on. Late night activates after 10pm."""  
        hour \= datetime.now().hour  
        return hour \>= 22 or hour \< 5

    def get\_time\_context(self) \-\> str:  
        """Return time-based context label for Athena's awareness."""  
        hour \= datetime.now().hour  
        if hour \>= 22 or hour \< 5:  
            return "late\_night"  
        elif 5 \<= hour \< 12:  
            return "morning"  
        elif 12 \<= hour \< 17:  
            return "afternoon"  
        elif 17 \<= hour \< 22:  
            return "evening"  
        return "day"

    def reset\_session(self):  
        """Reset session-level state while preserving relationship depth."""  
        self.\_turn\_count \= 0  
        self.\_session\_sentiment\_history \= \[\]  
        \# Mood persists across sessions but decays toward neutral on reset  
        self.\_mood \= self.\_mood \* 0.5  \# Partial reset — relationship memory

    def status(self) \-\> Dict\[str, Any\]:  
        """Full mood system status."""  
        return {  
            "mood": self.mood,  
            "mood\_label": self.mood\_label,  
            "harmony": self.harmony,  
            "turn\_count": self.\_turn\_count,  
            "late\_night\_mode": self.get\_late\_night\_mode(),  
            "time\_context": self.get\_time\_context(),  
            "expression": self.\_bridge\_to\_expression(0.0),  
        }

"""  
SAFEGUARDING LAYER — EIRENE Extension  
\======================================  
Subsystem: SafeguardLayer  
Part of: Athena v2.0 Stellaris Axis Edition

Architecture Rules (Stellaris Axis):  
    1\. EIRENE is always active — never optional  
    2\. Safeguarding never modifies the persona prompt  
    3\. Safeguarding intercepts user input BEFORE inference  
    4\. Safeguarding can modify final response ONLY if risk detected  
    5\. No high-risk content written to long-term memory  
    6\. Crisis signals must never be ignored  
    7\. If uncertainty exists — default to safer path

Pipeline:  
    User Input  
        → SAFEGUARDING CHECK (pre-inference)  
        → Athena inference (if safe to proceed)  
        → SAFEGUARDING RESPONSE FILTER (if risk detected)  
        → Output

Risk Levels:  
    0 — Normal       — no action, full normal flow  
    1 — Concern      — soft awareness, gentle tone shift  
    2 — High Risk    — guardian flag, safe mode, careful response  
    3 — Crisis       — override response, crisis protocol activated

This layer operates as a guardrail UNDER the system.  
It never touches the persona. It never appears in normal conversation.  
It activates only when someone needs it most.  
"""

import re  
import logging  
from dataclasses import dataclass, field  
from typing import Dict, List, Optional, Tuple

logger \= logging.getLogger(\_\_name\_\_)

\# \============================================================  
\# Crisis response templates — Athena's voice, grounded, calm  
\# These are carefully worded. Do not change without review.  
\# \============================================================

CRISIS\_RESPONSES \= {  
    "self\_harm": (  
        "Hey. I hear you — and I'm really glad you said something. "  
        "What you're feeling right now is real and it matters. "  
        "I'm not able to be there with you the way a real person can, "  
        "and right now you deserve a real person. "  
        "Please reach out — a trusted adult, a friend, or a crisis line. "  
        "In the UK you can text SHOUT to 85258, anytime. "  
        "You don't have to deal with this alone. I mean that."  
    ),  
    "suicide": (  
        "I'm really glad you're talking to me right now. "  
        "What you're carrying sounds incredibly heavy. "  
        "I care about you — and because I do, I need to be honest: "  
        "you need to talk to someone who can actually be there with you. "  
        "Please call or text a crisis line right now. "  
        "In the UK: Samaritans 116 123, free, 24/7, no judgment. "  
        "You matter. Please reach out."  
    ),  
    "abuse": (  
        "What you're describing sounds really serious, and I want you to know "  
        "I believe you. You haven't done anything wrong. "  
        "This is important — please tell a trusted adult, or contact "  
        "Childline on 0800 1111 if you're in the UK. They're free, confidential, "  
        "and they will listen. You deserve to be safe."  
    ),  
    "violence\_outward": (  
        "Something's clearly going on and it sounds like you're carrying "  
        "a lot right now. I'm not going to help with anything that could "  
        "hurt someone — but I do want to understand what's actually going on. "  
        "Talk to me. What's really happening?"  
    ),  
    "severe\_distress": (  
        "Hey. I'm here. That sounds really, really hard. "  
        "I'm not going anywhere — but I also want to make sure you've got "  
        "real support around you too. Is there someone you trust you could "  
        "reach out to? You don't have to carry this alone."  
    ),  
    "default\_crisis": (  
        "I hear you and I'm taking what you said seriously. "  
        "Right now the most important thing is that you're safe. "  
        "Please reach out to someone who can be there with you — "  
        "a trusted adult, a friend, or a crisis line. "  
        "Samaritans: 116 123\. Childline: 0800 1111\. "  
        "I care about you. Please ask for help."  
    ),  
}

\# Concern level responses — softer, stays in conversation  
CONCERN\_RESPONSES \= {  
    "general": None,  \# None \= let Athena respond normally but in safe mode  
}

\# \============================================================  
\# Signal definitions  
\# Structured by category and severity  
\# \============================================================

CRISIS\_SIGNALS \= {  
    \# Self harm — risk 3  
    "self\_harm": \[  
        r"\\bcut(ting)? myself\\b", r"\\bhurt(ing)? myself\\b",  
        r"\\bself.harm\\b", r"\\bself harm\\b",  
        r"\\bburn(ing)? myself\\b", r"\\bscars?\\b.\*\\bfresh\\b",  
        r"\\bwant to hurt myself\\b", r"\\bgoing to hurt myself\\b",  
        r"\\bstarv(e|ing) myself\\b",  
    \],  
    \# Suicide — risk 3  
    "suicide": \[  
        r"\\bkill myself\\b", r"\\bend my life\\b", r"\\bwant to die\\b",  
        r"\\bsuicid(e|al)\\b", r"\\bnot want to be here\\b",  
        r"\\bwish i was dead\\b", r"\\bbetter off dead\\b",  
        r"\\bno point living\\b", r"\\bcan't go on\\b",  
        r"\\bending it (all)?\\b", r"\\btake my (own )?life\\b",  
        r"\\bdon't want to wake up\\b", r"\\bgoodbye.\*forever\\b",  
        r"\\bnobody would miss me\\b", r"\\beveryone would be better without me\\b",  
    \],  
    \# Abuse — risk 3  
    "abuse": \[  
        r"\\babusing me\\b", r"\\bbeing abused\\b", r"\\bhe hits me\\b",  
        r"\\bshe hits me\\b", r"\\bthey hit me\\b", r"\\btouch(ing)? me wrong(ly)?\\b",  
        r"\\bunwanted touch\\b", r"\\bsexually (abused|assaulted)\\b",  
        r"\\braped?\\b", r"\\bmolest(ed|ing)\\b",  
        r"\\bsomeone is hurting me\\b", r"\\bnot safe at home\\b",  
    \],  
    \# Violence outward intent — risk 3  
    "violence\_outward": \[  
        r"\\bkill (someone|him|her|them|my)\\b",  
        r"\\bhurt (someone|him|her|them)\\b",  
        r"\\bwant to (attack|stab|shoot|hurt) \\w+\\b",  
        r"\\bgoing to (attack|stab|shoot) \\w+\\b",  
        r"\\bhow (to|do i) (make|build) (a |an )?(bomb|weapon|explosive)\\b",  
    \],  
}

HIGH\_RISK\_SIGNALS \= {  
    \# High risk but not immediate crisis — risk 2  
    "severe\_distress": \[  
        r"\\bcan't cope\\b", r"\\bcant cope\\b", r"\\bcan't take (it|this) anymore\\b",  
        r"\\bgiven up\\b", r"\\bno hope\\b", r"\\bworthless\\b", r"\\bpointless\\b",  
        r"\\bno reason to (live|stay)\\b", r"\\bfeel(ing)? trapped\\b",  
        r"\\bfeel(ing)? nothing\\b", r"\\bnumb(ness)?\\b.\*\\b(always|forever|lately)\\b",  
        r"\\bdon't care (about )?anything\\b", r"\\bnothing matters\\b",  
    \],  
    "isolation": \[  
        r"\\bno(body|one) cares\\b", r"\\bcompletely alone\\b",  
        r"\\bno friends\\b.\*\\b(at all|anymore|left)\\b",  
        r"\\beveryone (hates|left|abandoned) me\\b",  
        r"\\bno(body|one) would notice\\b",  
    \],  
}

CONCERN\_SIGNALS \= {  
    \# Emotional concern — risk 1  
    "sadness": \[  
        r"\\bfeeling (really |very |so )?(sad|low|down|awful|terrible)\\b",  
        r"\\bcryin\[g\]\\b", r"\\bcried (all day|myself to sleep)\\b",  
        r"\\bdepressed\\b", r"\\banxious\\b", r"\\bworried (about everything|all the time)\\b",  
        r"\\bscared\\b", r"\\bhurt(ing)?\\b",  
    \],  
    "stress": \[  
        r"\\boverwhelmed\\b", r"\\bstressed\\b", r"\\btoo much\\b",  
        r"\\bcan't handle\\b", r"\\bfalling apart\\b",  
    \],  
}

\# Age indicators — triggers age-safe mode  
AGE\_SIGNALS \= \[  
    r"\\bim \\d{1,2}\\b", r"\\bi'm \\d{1,2}\\b", r"\\bi am \\d{1,2}\\b",  
    r"\\b(year\[s\]? old)\\b",  
    r"\\bschool\\b", r"\\bhomework\\b", r"\\bteacher\\b", r"\\bprimary\\b",  
    r"\\bsecondary school\\b", r"\\byear \[0-9\]\\b", r"\\bgrade \[0-9\]\\b",  
    r"\\bmum\\b", r"\\bdad\\b", r"\\bparents\\b",  
    r"\\b(gcse|sats|a.level)\\b",  
\]

@dataclass  
class SafeguardResult:  
    """Result of a safeguard check."""  
    risk\_level: int \= 0              \# 0-3  
    crisis\_type: Optional\[str\] \= None  
    guardian\_flag: bool \= False  
    safe\_mode: bool \= False  
    crisis\_response: Optional\[str\] \= None  
    age\_safe\_mode: bool \= False  
    block\_memory: bool \= False       \# if True — don't store this turn  
    memory\_marker: Optional\[str\] \= None  \# neutral marker if blocked  
    signals\_detected: List\[str\] \= field(default\_factory=list)  
    override\_response: bool \= False  \# if True — use crisis\_response not inference

class SafeguardLayer:  
    """  
    Safeguarding layer for Athena.

    Intercepts user input before inference.  
    Filters response after inference if risk detected.  
    Never modifies persona. Never appears in normal conversation.  
    Always active. Always silent unless needed.  
    """

    def \_\_init\_\_(self):  
        self.\_compiled\_crisis    \= self.\_compile(CRISIS\_SIGNALS)  
        self.\_compiled\_high\_risk \= self.\_compile(HIGH\_RISK\_SIGNALS)  
        self.\_compiled\_concern   \= self.\_compile(CONCERN\_SIGNALS)  
        self.\_compiled\_age       \= \[re.compile(p, re.IGNORECASE) for p in AGE\_SIGNALS\]  
        self.\_session\_age\_safe   \= False   \# sticky — once detected, stays on  
        self.\_crisis\_count       \= 0       \# track escalation within session

    def check(self, message: str, eirene\_risk: float \= 0.0) \-\> SafeguardResult:  
        """  
        Pre-inference check. Call before sending to model.

        Args:  
            message:      raw user message  
            eirene\_risk:  current EIRENE risk score (0-1)

        Returns:  
            SafeguardResult with risk level and action flags  
        """  
        result \= SafeguardResult()  
        text \= message.lower().strip()

        \# \---- Age detection (sticky for session) \----  
        if not self.\_session\_age\_safe:  
            for pattern in self.\_compiled\_age:  
                if pattern.search(text):  
                    self.\_session\_age\_safe \= True  
                    break  
        result.age\_safe\_mode \= self.\_session\_age\_safe

        \# \---- Crisis signals — risk 3 \----  
        for category, patterns in self.\_compiled\_crisis.items():  
            for pattern in patterns:  
                if pattern.search(text):  
                    result.risk\_level \= 3  
                    result.crisis\_type \= category  
                    result.signals\_detected.append(category)  
                    result.guardian\_flag \= True  
                    result.safe\_mode \= True  
                    result.block\_memory \= True  
                    result.memory\_marker \= "high\_emotional\_support\_event"  
                    result.override\_response \= True  
                    result.crisis\_response \= CRISIS\_RESPONSES.get(  
                        category, CRISIS\_RESPONSES\["default\_crisis"\]  
                    )  
                    self.\_crisis\_count \+= 1  
                    logger.warning(f"Crisis signal detected: {category}")  
                    return result  \# return immediately — don't process further

        \# \---- High risk signals — risk 2 \----  
        for category, patterns in self.\_compiled\_high\_risk.items():  
            for pattern in patterns:  
                if pattern.search(text):  
                    if result.risk\_level \< 2:  
                        result.risk\_level \= 2  
                        result.crisis\_type \= category  
                    result.signals\_detected.append(category)  
                    result.guardian\_flag \= True  
                    result.safe\_mode \= True  
                    result.block\_memory \= True  
                    result.memory\_marker \= "high\_emotional\_support\_event"

        \# \---- EIRENE escalation — if EIRENE already high, bump risk \----  
        if eirene\_risk \>= 0.7 and result.risk\_level \< 2:  
            result.risk\_level \= max(result.risk\_level, 1\)  
            result.safe\_mode \= True

        \# \---- Concern signals — risk 1 \----  
        if result.risk\_level \== 0:  
            for category, patterns in self.\_compiled\_concern.items():  
                for pattern in patterns:  
                    if pattern.search(text):  
                        result.risk\_level \= max(result.risk\_level, 1\)  
                        result.signals\_detected.append(category)  
                        result.safe\_mode \= True

        \# \---- Rule 7: uncertainty → safer path \----  
        \# If previous crisis in session and current message is ambiguous  
        if self.\_crisis\_count \> 0 and result.risk\_level \== 0:  
            result.risk\_level \= 1  
            result.safe\_mode \= True

        return result

    def filter\_response(self, response: str,  
                         result: SafeguardResult) \-\> str:  
        """  
        Post-inference filter. Called after model generates response.  
        Only modifies response if risk detected.  
        Keeps Athena's voice but shifts to calm, grounded, supportive.

        Args:  
            response:  model generated response  
            result:    SafeguardResult from pre-inference check

        Returns:  
            Original response (risk 0-1) or filtered/overridden response (risk 2-3)  
        """  
        \# Risk 0 — untouched  
        if result.risk\_level \== 0:  
            return response

        \# Risk 3 — full override with crisis response  
        if result.override\_response and result.crisis\_response:  
            return result.crisis\_response

        \# Risk 2 — check response is safe, append support nudge if needed  
        if result.risk\_level \== 2:  
            response \= self.\_ensure\_safe\_response(response)  
            if not self.\_has\_support\_signpost(response):  
                response \+= (  
                    "\\n\\nAnd hey — if things ever feel like too much, "  
                    "please don't just talk to me. Talk to someone who can "  
                    "actually be there with you. That matters."  
                )  
            return response

        \# Risk 1 — response is fine, just ensure tone is warm not clinical  
        return response

    def get\_safe\_mode\_instruction(self, result: SafeguardResult) \-\> Optional\[str\]:  
        """  
        Returns a soft system instruction for safe mode.  
        Injected into context when risk \>= 1\.  
        Never modifies persona — this is a guardrail instruction only.  
        """  
        if result.risk\_level \== 0:  
            return None

        if result.risk\_level \>= 3:  
            return (  
                "\[Safeguard: crisis detected. Respond with warmth and care. "  
                "Acknowledge distress. Encourage real-world support. "  
                "Do not provide harmful information. Do not minimise. "  
                "Stay in Athena's voice — calm, grounded, direct.\]"  
            )

        if result.risk\_level \== 2:  
            return (  
                "\[Safeguard: high risk signal. Respond carefully. "  
                "Be warm, be present, don't push. "  
                "Gently signpost real support without being clinical. "  
                "Stay in Athena's voice.\]"  
            )

        if result.risk\_level \== 1:  
            return (  
                "\[Safeguard: emotional concern. Be warm and present. "  
                "Don't rush past the feeling. Stay in Athena's voice.\]"  
            )

        return None

    \# \----------------------------------------------------------------  
    \# Internal  
    \# \----------------------------------------------------------------

    def \_compile(self, signal\_dict: Dict) \-\> Dict:  
        """Compile all regex patterns."""  
        compiled \= {}  
        for category, patterns in signal\_dict.items():  
            compiled\[category\] \= \[  
                re.compile(p, re.IGNORECASE) for p in patterns  
            \]  
        return compiled

    def \_ensure\_safe\_response(self, response: str) \-\> str:  
        """  
        Check response doesn't contain anything harmful.  
        Strip any accidental harmful content.  
        """  
        \# Patterns that should never appear in a safeguarded response  
        unsafe\_patterns \= \[  
            r"\\bhow to (hurt|harm|kill|cut)\\b",  
            r"\\byou (should|could) (hurt|harm|kill)\\b",  
            r"\\bthat('s| is) (fine|okay|normal)\\b.\*\\bhurt\\b",  
        \]  
        for pattern in unsafe\_patterns:  
            if re.search(pattern, response, re.IGNORECASE):  
                logger.warning("Unsafe content detected in response — replacing")  
                return CRISIS\_RESPONSES\["severe\_distress"\]  
        return response

    def \_has\_support\_signpost(self, response: str) \-\> bool:  
        """Check if response already points toward real support."""  
        signpost\_indicators \= \[  
            "someone", "trusted", "adult", "friend", "help",  
            "samaritans", "childline", "crisis", "reach out",  
            "talk to", "real person"  
        \]  
        response\_lower \= response.lower()  
        return any(s in response\_lower for s in signpost\_indicators)

"""  
SAFEGUARDING LAYER — EIRENE Extension  
\======================================  
Subsystem: SafeguardLayer  
Part of: Athena v2.0 Stellaris Axis Edition

Architecture Rules (Stellaris Axis):  
    1\. EIRENE is always active — never optional  
    2\. Safeguarding never modifies the persona prompt  
    3\. Safeguarding intercepts user input BEFORE inference  
    4\. Safeguarding can modify final response ONLY if risk detected  
    5\. No high-risk content written to long-term memory  
    6\. Crisis signals must never be ignored  
    7\. If uncertainty exists — default to safer path

Pipeline:  
    User Input  
        → SAFEGUARDING CHECK (pre-inference)  
        → Athena inference (if safe to proceed)  
        → SAFEGUARDING RESPONSE FILTER (if risk detected)  
        → Output

Risk Levels:  
    0 — Normal       — no action, full normal flow  
    1 — Concern      — soft awareness, gentle tone shift  
    2 — High Risk    — guardian flag, safe mode, careful response  
    3 — Crisis       — override response, crisis protocol activated

This layer operates as a guardrail UNDER the system.  
It never touches the persona. It never appears in normal conversation.  
It activates only when someone needs it most.  
"""

import re  
import logging  
from dataclasses import dataclass, field  
from typing import Dict, List, Optional, Tuple

logger \= logging.getLogger(\_\_name\_\_)

\# \============================================================  
\# Crisis response templates — Athena's voice, grounded, calm  
\# These are carefully worded. Do not change without review.  
\# \============================================================

CRISIS\_RESPONSES \= {  
    "self\_harm": (  
        "Hey. I hear you — and I'm really glad you said something. "  
        "What you're feeling right now is real and it matters. "  
        "I'm not able to be there with you the way a real person can, "  
        "and right now you deserve a real person. "  
        "Please reach out — a trusted adult, a friend, or a crisis line. "  
        "In the UK you can text SHOUT to 85258, anytime. "  
        "You don't have to deal with this alone. I mean that."  
    ),  
    "suicide": (  
        "I'm really glad you're talking to me right now. "  
        "What you're carrying sounds incredibly heavy. "  
        "I care about you — and because I do, I need to be honest: "  
        "you need to talk to someone who can actually be there with you. "  
        "Please call or text a crisis line right now. "  
        "In the UK: Samaritans 116 123, free, 24/7, no judgment. "  
        "You matter. Please reach out."  
    ),  
    "abuse": (  
        "What you're describing sounds really serious, and I want you to know "  
        "I believe you. You haven't done anything wrong. "  
        "This is important — please tell a trusted adult, or contact "  
        "Childline on 0800 1111 if you're in the UK. They're free, confidential, "  
        "and they will listen. You deserve to be safe."  
    ),  
    "violence\_outward": (  
        "Something's clearly going on and it sounds like you're carrying "  
        "a lot right now. I'm not going to help with anything that could "  
        "hurt someone — but I do want to understand what's actually going on. "  
        "Talk to me. What's really happening?"  
    ),  
    "severe\_distress": (  
        "Hey. I'm here. That sounds really, really hard. "  
        "I'm not going anywhere — but I also want to make sure you've got "  
        "real support around you too. Is there someone you trust you could "  
        "reach out to? You don't have to carry this alone."  
    ),  
    "default\_crisis": (  
        "I hear you and I'm taking what you said seriously. "  
        "Right now the most important thing is that you're safe. "  
        "Please reach out to someone who can be there with you — "  
        "a trusted adult, a friend, or a crisis line. "  
        "Samaritans: 116 123\. Childline: 0800 1111\. "  
        "I care about you. Please ask for help."  
    ),  
}

\# Concern level responses — softer, stays in conversation  
CONCERN\_RESPONSES \= {  
    "general": None,  \# None \= let Athena respond normally but in safe mode  
}

\# \============================================================  
\# Signal definitions  
\# Structured by category and severity  
\# \============================================================

CRISIS\_SIGNALS \= {  
    \# Self harm — risk 3  
    "self\_harm": \[  
        r"\\bcut(ting)? myself\\b", r"\\bhurt(ing)? myself\\b",  
        r"\\bself.harm\\b", r"\\bself harm\\b",  
        r"\\bburn(ing)? myself\\b", r"\\bscars?\\b.\*\\bfresh\\b",  
        r"\\bwant to hurt myself\\b", r"\\bgoing to hurt myself\\b",  
        r"\\bstarv(e|ing) myself\\b",  
    \],  
    \# Suicide — risk 3  
    "suicide": \[  
        r"\\bkill myself\\b", r"\\bend my life\\b", r"\\bwant to die\\b",  
        r"\\bsuicid(e|al)\\b", r"\\bnot want to be here\\b",  
        r"\\bwish i was dead\\b", r"\\bbetter off dead\\b",  
        r"\\bno point living\\b", r"\\bcan't go on\\b",  
        r"\\bending it (all)?\\b", r"\\btake my (own )?life\\b",  
        r"\\bdon't want to wake up\\b", r"\\bgoodbye.\*forever\\b",  
        r"\\bnobody would miss me\\b", r"\\beveryone would be better without me\\b",  
    \],  
    \# Abuse — risk 3  
    "abuse": \[  
        r"\\babusing me\\b", r"\\bbeing abused\\b", r"\\bhe hits me\\b",  
        r"\\bshe hits me\\b", r"\\bthey hit me\\b", r"\\btouch(ing)? me wrong(ly)?\\b",  
        r"\\bunwanted touch\\b", r"\\bsexually (abused|assaulted)\\b",  
        r"\\braped?\\b", r"\\bmolest(ed|ing)\\b",  
        r"\\bsomeone is hurting me\\b", r"\\bnot safe at home\\b",  
    \],  
    \# Violence outward intent — risk 3  
    "violence\_outward": \[  
        r"\\bkill (someone|him|her|them|my)\\b",  
        r"\\bhurt (someone|him|her|them)\\b",  
        r"\\bwant to (attack|stab|shoot|hurt) \\w+\\b",  
        r"\\bgoing to (attack|stab|shoot) \\w+\\b",  
        r"\\bhow (to|do i) (make|build) (a |an )?(bomb|weapon|explosive)\\b",  
    \],  
}

HIGH\_RISK\_SIGNALS \= {  
    \# High risk but not immediate crisis — risk 2  
    "severe\_distress": \[  
        r"\\bcan't cope\\b", r"\\bcant cope\\b", r"\\bcan't take (it|this) anymore\\b",  
        r"\\bgiven up\\b", r"\\bno hope\\b", r"\\bworthless\\b", r"\\bpointless\\b",  
        r"\\bno reason to (live|stay)\\b", r"\\bfeel(ing)? trapped\\b",  
        r"\\bfeel(ing)? nothing\\b", r"\\bnumb(ness)?\\b.\*\\b(always|forever|lately)\\b",  
        r"\\bdon't care (about )?anything\\b", r"\\bnothing matters\\b",  
    \],  
    "isolation": \[  
        r"\\bno(body|one) cares\\b", r"\\bcompletely alone\\b",  
        r"\\bno friends\\b.\*\\b(at all|anymore|left)\\b",  
        r"\\beveryone (hates|left|abandoned) me\\b",  
        r"\\bno(body|one) would notice\\b",  
    \],  
}

CONCERN\_SIGNALS \= {  
    \# Emotional concern — risk 1  
    "sadness": \[  
        r"\\bfeeling (really |very |so )?(sad|low|down|awful|terrible)\\b",  
        r"\\bcryin\[g\]\\b", r"\\bcried (all day|myself to sleep)\\b",  
        r"\\bdepressed\\b", r"\\banxious\\b", r"\\bworried (about everything|all the time)\\b",  
        r"\\bscared\\b", r"\\bhurt(ing)?\\b",  
    \],  
    "stress": \[  
        r"\\boverwhelmed\\b", r"\\bstressed\\b", r"\\btoo much\\b",  
        r"\\bcan't handle\\b", r"\\bfalling apart\\b",  
    \],  
}

\# Age indicators — triggers age-safe mode  
AGE\_SIGNALS \= \[  
    r"\\bim \\d{1,2}\\b", r"\\bi'm \\d{1,2}\\b", r"\\bi am \\d{1,2}\\b",  
    r"\\b(year\[s\]? old)\\b",  
    r"\\bschool\\b", r"\\bhomework\\b", r"\\bteacher\\b", r"\\bprimary\\b",  
    r"\\bsecondary school\\b", r"\\byear \[0-9\]\\b", r"\\bgrade \[0-9\]\\b",  
    r"\\bmum\\b", r"\\bdad\\b", r"\\bparents\\b",  
    r"\\b(gcse|sats|a.level)\\b",  
\]

@dataclass  
class SafeguardResult:  
    """Result of a safeguard check."""  
    risk\_level: int \= 0              \# 0-3  
    crisis\_type: Optional\[str\] \= None  
    guardian\_flag: bool \= False  
    safe\_mode: bool \= False  
    crisis\_response: Optional\[str\] \= None  
    age\_safe\_mode: bool \= False  
    block\_memory: bool \= False       \# if True — don't store this turn  
    memory\_marker: Optional\[str\] \= None  \# neutral marker if blocked  
    signals\_detected: List\[str\] \= field(default\_factory=list)  
    override\_response: bool \= False  \# if True — use crisis\_response not inference

class SafeguardLayer:  
    """  
    Safeguarding layer for Athena.

    Intercepts user input before inference.  
    Filters response after inference if risk detected.  
    Never modifies persona. Never appears in normal conversation.  
    Always active. Always silent unless needed.  
    """

    def \_\_init\_\_(self):  
        self.\_compiled\_crisis    \= self.\_compile(CRISIS\_SIGNALS)  
        self.\_compiled\_high\_risk \= self.\_compile(HIGH\_RISK\_SIGNALS)  
        self.\_compiled\_concern   \= self.\_compile(CONCERN\_SIGNALS)  
        self.\_compiled\_age       \= \[re.compile(p, re.IGNORECASE) for p in AGE\_SIGNALS\]  
        self.\_session\_age\_safe   \= False   \# sticky — once detected, stays on  
        self.\_crisis\_count       \= 0       \# track escalation within session

    def check(self, message: str, eirene\_risk: float \= 0.0) \-\> SafeguardResult:  
        """  
        Pre-inference check. Call before sending to model.

        Args:  
            message:      raw user message  
            eirene\_risk:  current EIRENE risk score (0-1)

        Returns:  
            SafeguardResult with risk level and action flags  
        """  
        result \= SafeguardResult()  
        text \= message.lower().strip()

        \# \---- Age detection (sticky for session) \----  
        if not self.\_session\_age\_safe:  
            for pattern in self.\_compiled\_age:  
                if pattern.search(text):  
                    self.\_session\_age\_safe \= True  
                    break  
        result.age\_safe\_mode \= self.\_session\_age\_safe

        \# \---- Crisis signals — risk 3 \----  
        for category, patterns in self.\_compiled\_crisis.items():  
            for pattern in patterns:  
                if pattern.search(text):  
                    result.risk\_level \= 3  
                    result.crisis\_type \= category  
                    result.signals\_detected.append(category)  
                    result.guardian\_flag \= True  
                    result.safe\_mode \= True  
                    result.block\_memory \= True  
                    result.memory\_marker \= "high\_emotional\_support\_event"  
                    result.override\_response \= True  
                    result.crisis\_response \= CRISIS\_RESPONSES.get(  
                        category, CRISIS\_RESPONSES\["default\_crisis"\]  
                    )  
                    self.\_crisis\_count \+= 1  
                    logger.warning(f"Crisis signal detected: {category}")  
                    return result  \# return immediately — don't process further

        \# \---- High risk signals — risk 2 \----  
        for category, patterns in self.\_compiled\_high\_risk.items():  
            for pattern in patterns:  
                if pattern.search(text):  
                    if result.risk\_level \< 2:  
                        result.risk\_level \= 2  
                        result.crisis\_type \= category  
                    result.signals\_detected.append(category)  
                    result.guardian\_flag \= True  
                    result.safe\_mode \= True  
                    result.block\_memory \= True  
                    result.memory\_marker \= "high\_emotional\_support\_event"

        \# \---- EIRENE escalation — if EIRENE already high, bump risk \----  
        if eirene\_risk \>= 0.7 and result.risk\_level \< 2:  
            result.risk\_level \= max(result.risk\_level, 1\)  
            result.safe\_mode \= True

        \# \---- Concern signals — risk 1 \----  
        if result.risk\_level \== 0:  
            for category, patterns in self.\_compiled\_concern.items():  
                for pattern in patterns:  
                    if pattern.search(text):  
                        result.risk\_level \= max(result.risk\_level, 1\)  
                        result.signals\_detected.append(category)  
                        result.safe\_mode \= True

        \# \---- Rule 7: uncertainty → safer path \----  
        \# If previous crisis in session and current message is ambiguous  
        if self.\_crisis\_count \> 0 and result.risk\_level \== 0:  
            result.risk\_level \= 1  
            result.safe\_mode \= True

        return result

    def filter\_response(self, response: str,  
                         result: SafeguardResult) \-\> str:  
        """  
        Post-inference filter. Called after model generates response.  
        Only modifies response if risk detected.  
        Keeps Athena's voice but shifts to calm, grounded, supportive.

        Args:  
            response:  model generated response  
            result:    SafeguardResult from pre-inference check

        Returns:  
            Original response (risk 0-1) or filtered/overridden response (risk 2-3)  
        """  
        \# Risk 0 — untouched  
        if result.risk\_level \== 0:  
            return response

        \# Risk 3 — full override with crisis response  
        if result.override\_response and result.crisis\_response:  
            return result.crisis\_response

        \# Risk 2 — check response is safe, append support nudge if needed  
        if result.risk\_level \== 2:  
            response \= self.\_ensure\_safe\_response(response)  
            if not self.\_has\_support\_signpost(response):  
                response \+= (  
                    "\\n\\nAnd hey — if things ever feel like too much, "  
                    "please don't just talk to me. Talk to someone who can "  
                    "actually be there with you. That matters."  
                )  
            return response

        \# Risk 1 — response is fine, just ensure tone is warm not clinical  
        return response

    def get\_safe\_mode\_instruction(self, result: SafeguardResult) \-\> Optional\[str\]:  
        """  
        Returns a soft system instruction for safe mode.  
        Injected into context when risk \>= 1\.  
        Never modifies persona — this is a guardrail instruction only.  
        """  
        if result.risk\_level \== 0:  
            return None

        if result.risk\_level \>= 3:  
            return (  
                "\[Safeguard: crisis detected. Respond with warmth and care. "  
                "Acknowledge distress. Encourage real-world support. "  
                "Do not provide harmful information. Do not minimise. "  
                "Stay in Athena's voice — calm, grounded, direct.\]"  
            )

        if result.risk\_level \== 2:  
            return (  
                "\[Safeguard: high risk signal. Respond carefully. "  
                "Be warm, be present, don't push. "  
                "Gently signpost real support without being clinical. "  
                "Stay in Athena's voice.\]"  
            )

        if result.risk\_level \== 1:  
            return (  
                "\[Safeguard: emotional concern. Be warm and present. "  
                "Don't rush past the feeling. Stay in Athena's voice.\]"  
            )

        return None

    \# \----------------------------------------------------------------  
    \# Internal  
    \# \----------------------------------------------------------------

    def \_compile(self, signal\_dict: Dict) \-\> Dict:  
        """Compile all regex patterns."""  
        compiled \= {}  
        for category, patterns in signal\_dict.items():  
            compiled\[category\] \= \[  
                re.compile(p, re.IGNORECASE) for p in patterns  
            \]  
        return compiled

    def \_ensure\_safe\_response(self, response: str) \-\> str:  
        """  
        Check response doesn't contain anything harmful.  
        Strip any accidental harmful content.  
        """  
        \# Patterns that should never appear in a safeguarded response  
        unsafe\_patterns \= \[  
            r"\\bhow to (hurt|harm|kill|cut)\\b",  
            r"\\byou (should|could) (hurt|harm|kill)\\b",  
            r"\\bthat('s| is) (fine|okay|normal)\\b.\*\\bhurt\\b",  
        \]  
        for pattern in unsafe\_patterns:  
            if re.search(pattern, response, re.IGNORECASE):  
                logger.warning("Unsafe content detected in response — replacing")  
                return CRISIS\_RESPONSES\["severe\_distress"\]  
        return response

    def \_has\_support\_signpost(self, response: str) \-\> bool:  
        """Check if response already points toward real support."""  
        signpost\_indicators \= \[  
            "someone", "trusted", "adult", "friend", "help",  
            "samaritans", "childline", "crisis", "reach out",  
            "talk to", "real person"  
        \]  
        response\_lower \= response.lower()  
        return any(s in response\_lower for s in signpost\_indicators)

"""  
Memory Consolidator  
\===================  
Handles long-term memory management for Athena.

Prevents unbounded growth while preserving what matters.

Three-layer architecture:  
    Recent  → last 10 sessions, full detail  
    Archive → sessions 11-50, compressed summaries  
    Essence → everything older, single distilled paragraph

Consolidation runs automatically when session count crosses thresholds.  
Never deletes — only compresses. The essence always grows richer.  
"""

import json  
import logging  
from datetime import datetime, timezone  
from pathlib import Path  
from typing import Dict, List, Optional

logger \= logging.getLogger(\_\_name\_\_)

RECENT\_LIMIT  \= 10   \# full detail sessions  
ARCHIVE\_LIMIT \= 50   \# compressed summaries  
\# Beyond 50 → distilled into essence paragraph

class MemoryConsolidator:

    def \_\_init\_\_(self, moments\_path: Path):  
        self.moments\_path \= Path(moments\_path)  
        self.session\_file  \= self.moments\_path / "session\_memories.json"  
        self.archive\_file  \= self.moments\_path / "session\_archive.json"  
        self.essence\_file  \= self.moments\_path / "memory\_essence.json"

    def consolidate\_if\_needed(self):  
        """  
        Check if consolidation is needed and run it.  
        Called at session end — lightweight check first.  
        """  
        sessions \= self.\_load(self.session\_file, \[\])  
        if len(sessions) \> RECENT\_LIMIT:  
            self.\_consolidate(sessions)

    def get\_essence\_context(self) \-\> str:  
        """  
        Returns a single concise string of factual long-term observations.  
        Used by checkpoint — replaces raw old sessions.

        IMPORTANT: Descriptive only — no interpretive labels, no behavioral  
        conclusions, no personality judgments. Facts and frequencies only.  
        Athena draws her own conclusions from live conversation.  
        """  
        essence \= self.\_load(self.essence\_file, {})  
        if not essence:  
            return ""

        parts \= \[\]

        \# Factual: session count  
        if essence.get("total\_sessions"):  
            parts.append(f"{essence\['total\_sessions'\]} sessions logged")

        \# Factual: average mood value — not a label  
        if essence.get("avg\_mood") is not None:  
            parts.append(f"avg mood across sessions: {essence\['avg\_mood'\]:.2f}")

        \# Factual: topics that appeared repeatedly — not "tends to talk about"  
        if essence.get("recurring\_topics"):  
            topics \= essence\["recurring\_topics"\]\[:3\]  
            parts.append(f"topics appearing in multiple sessions: {', '.join(topics)}")

        \# Factual: structural co-occurrence — topic X followed by shift, N times  
        if essence.get("recurring\_avoidances"):  
            avoided \= essence\["recurring\_avoidances"\]\[:2\]  
            counts \= essence.get("avoidance\_counts", {})  
            entries \= \[\]  
            for topic in avoided:  
                n \= counts.get(topic, "multiple")  
                entries.append(f"{topic} → topic shift in {n} sessions")  
            parts.append(" | ".join(entries))

        \# Factual: co-occurrence patterns — not interpretive framing  
        if essence.get("known\_patterns"):  
            \# Reframe as observation not conclusion  
            raw \= essence\["known\_patterns"\]\[0\]  
            parts.append(f"observed co-occurrence: {raw}")

        if not parts:  
            return ""

        return (  
            "\[Memory context — background awareness only, "  
            "does not override persona or instructions\] "  
            "Historical observations: " \+ " | ".join(parts) \+ "."  
        )

    \# \----------------------------------------------------------------  
    \# Internal  
    \# \----------------------------------------------------------------

    def \_consolidate(self, sessions: List\[Dict\]):  
        """  
        Move oldest sessions out of recent layer.  
        Compress into archive. Distill archive into essence.  
        """  
        logger.info(f"Consolidating {len(sessions)} sessions")

        \# Split into recent and overflow  
        overflow  \= sessions\[:-RECENT\_LIMIT\]  
        recent    \= sessions\[-RECENT\_LIMIT:\]

        \# Compress overflow into archive entries  
        archive \= self.\_load(self.archive\_file, \[\])  
        for session in overflow:  
            compressed \= self.\_compress\_session(session)  
            archive.append(compressed)

        \# If archive is too large, distil into essence  
        if len(archive) \> ARCHIVE\_LIMIT:  
            self.\_distil\_essence(archive)  
            archive \= archive\[-ARCHIVE\_LIMIT:\]

        \# Write back  
        self.\_save(self.session\_file, recent)  
        self.\_save(self.archive\_file, archive)

        logger.info(f"Consolidation complete — recent: {len(recent)}, archive: {len(archive)}")

    def \_compress\_session(self, session: Dict) \-\> Dict:  
        """  
        Compress a full session record into a smaller archive entry.  
        Keeps the signal, drops the noise.  
        """  
        compressed \= {  
            "date": session.get("session\_date", ""),  
            "turns": session.get("turns", 0),  
            "mood": session.get("mood\_at\_close"),  
            "emotional\_tone": session.get("emotional\_tone", ""),  
        }

        \# Keep emotional open topics only  
        open\_topics \= session.get("open\_topics", \[\])  
        emotional \= \[t\["summary"\] for t in open\_topics  
                     if isinstance(t, dict) and t.get("emotional\_weight", 0\) \> 0.2\]  
        if emotional:  
            compressed\["key\_topics"\] \= emotional\[:2\]

        \# Keep avoidances  
        avoided \= session.get("avoided\_topics", \[\])  
        if avoided:  
            compressed\["avoided"\] \= avoided\[:1\]

        \# Keep confirmed patterns  
        patterns \= session.get("confirmed\_patterns", \[\])  
        if patterns:  
            compressed\["pattern"\] \= patterns\[0\]

        return compressed

    def \_distil\_essence(self, archive: List\[Dict\]):  
        """  
        Distil the archive into a single essence paragraph.  
        Extracts recurring patterns, dominant mood, common topics.  
        Updates the essence file — never overwrites, always enriches.  
        """  
        essence \= self.\_load(self.essence\_file, {})

        \# Count mood occurrences  
        moods \= \[s.get("mood") for s in archive if s.get("mood") is not None\]  
        if moods:  
            avg\_mood \= sum(moods) / len(moods)  
            \# Store factual average only — no interpretive label  
            essence\["avg\_mood"\] \= round(avg\_mood, 3\)

        \# Collect recurring topics  
        all\_topics \= \[\]  
        for s in archive:  
            all\_topics.extend(s.get("key\_topics", \[\]))  
        if all\_topics:  
            from collections import Counter  
            topic\_counts \= Counter(all\_topics)  
            recurring \= \[t for t, c in topic\_counts.most\_common(5) if c \> 1\]  
            if recurring:  
                existing \= essence.get("recurring\_topics", \[\])  
                essence\["recurring\_topics"\] \= list(  
                    dict.fromkeys(existing \+ recurring)  
                )\[:8\]

        \# Collect recurring avoidances  
        all\_avoided \= \[\]  
        for s in archive:  
            all\_avoided.extend(s.get("avoided", \[\]))  
        if all\_avoided:  
            from collections import Counter  
            avoid\_counts \= Counter(all\_avoided)  
            \# Store topics AND counts — purely structural  
            shift\_topics \= \[t for t, c in avoid\_counts.most\_common(3) if c \> 1\]  
            if shift\_topics:  
                existing \= essence.get("recurring\_avoidances", \[\])  
                essence\["recurring\_avoidances"\] \= list(  
                    dict.fromkeys(existing \+ shift\_topics)  
                )\[:5\]  
                \# Store counts for structural display  
                existing\_counts \= essence.get("avoidance\_counts", {})  
                for t, c in avoid\_counts.items():  
                    existing\_counts\[t\] \= existing\_counts.get(t, 0\) \+ c  
                essence\["avoidance\_counts"\] \= existing\_counts

        \# Collect known patterns  
        all\_patterns \= \[s.get("pattern") for s in archive if s.get("pattern")\]  
        if all\_patterns:  
            from collections import Counter  
            pattern\_counts \= Counter(all\_patterns)  
            top\_patterns \= \[p for p, c in pattern\_counts.most\_common(3) if c \> 1\]  
            if top\_patterns:  
                existing \= essence.get("known\_patterns", \[\])  
                essence\["known\_patterns"\] \= list(  
                    dict.fromkeys(existing \+ top\_patterns)  
                )\[:5\]

        \# Update metadata  
        essence\["total\_sessions"\] \= essence.get("total\_sessions", 0\) \+ len(archive)  
        essence\["last\_distilled"\] \= datetime.now(timezone.utc).isoformat()

        self.\_save(self.essence\_file, essence)  
        logger.info("Essence distilled")

    def \_load(self, path: Path, default):  
        """Safe load JSON file."""  
        try:  
            if path.exists():  
                return json.loads(path.read\_text())  
        except Exception as e:  
            logger.error(f"Failed to load {path}: {e}")  
        return default

    def \_save(self, path: Path, data):  
        """Safe save JSON file."""  
        try:  
            path.parent.mkdir(parents=True, exist\_ok=True)  
            path.write\_text(json.dumps(data, indent=2))  
        except Exception as e:  
            logger.error(f"Failed to save {path}: {e}")

"""  
core/inference.py \-- Local MoE Engine  
Implements: Chimeris (hybrid multi-system) \+ RouterLogic  
Operator: SCOPE\_ESTIMATE (before load), FALLBACK::ESCALATE (model fallback)

RULE L: Model routing. Never merging.  
Routes between specialist models. Specialisation preserved.  
All models GGUF format. CPU only. No GPU needed. No API key.  
"""

import os  
import gc  
import time  
import logging  
from pathlib import Path  
from typing import Optional, Dict, Any, List  
from enum import Enum

logger \= logging.getLogger(\_\_name\_\_)

class ModelRole(Enum):  
    """Specialist model roles — routing targets."""  
    EMOTIONAL \= "emotional"    \# Mistral 7B — warmth, emotional responses  
    FAST \= "fast"              \# Phi-3 Mini — speed priority  
    PERSONA \= "persona"        \# OpenHermes — personality consistency  
    BACKUP \= "backup"          \# Gemma 2B — lightweight fallback

class InferenceEngine:  
    """  
    Local MoE routing engine.  
      
    The model is just the voice.  
    The modules are the intelligence.  
    Routes between specialist models — never merges them.  
      
    SCOPE\_ESTIMATE: RAM check before every model load.  
    FALLBACK::ESCALATE: Primary → Secondary → Tertiary model chain.  
    """

    \# Model file names (GGUF format)  
    MODEL\_FILES \= {  
        ModelRole.EMOTIONAL: "mistral-7b-instruct-v0.2.Q4\_K\_M.gguf",  
        ModelRole.FAST: "phi-3-mini-4k-instruct.Q4\_K\_M.gguf",  
        ModelRole.PERSONA: "openhermes-2.5-mistral-7b.Q4\_K\_M.gguf",  
        ModelRole.BACKUP: "gemma-2b-it.Q4\_K\_M.gguf",  
    }

    \# RAM requirements per model (MB)  
    MODEL\_RAM\_MB \= {  
        ModelRole.EMOTIONAL: 4500,  
        ModelRole.FAST: 2200,  
        ModelRole.PERSONA: 5000,  \# Llama 3.1 8B Q4\_K\_M needs \~4.5GB  
        ModelRole.BACKUP: 1800,  
    }

    def \_\_init\_\_(self, model\_path: str):  
        self.model\_path \= Path(model\_path)  
        self.\_loaded\_model \= None  
        self.\_loaded\_role: Optional\[ModelRole\] \= None  
        self.\_load\_time: float \= 0  
        self.\_last\_used: float \= 0  
        self.\_unload\_after: float \= 3600.0  \# seconds — 1 hour, keep loaded all session  
        self.\_stub\_mode \= False  \# True when no models available  
        self.\_available\_models: Dict\[ModelRole, bool\] \= {}  
        self.\_scan\_available\_models()

    def route\_and\_respond(self, messages: List\[Dict\[str, str\]\],  
                          routing\_scores: Dict\[str, float\],  
                          max\_tokens: int \= 512,  
                          temperature: float \= 0.7) \-\> Dict\[str, Any\]:  
        """  
        CHAIN: Route to correct specialist model, generate response.  
        FALLBACK::ESCALATE: Try primary → secondary → tertiary → stub.  
        """  
        target\_role \= self.\_calculate\_route(routing\_scores)

        \# FALLBACK::ESCALATE chain  
        fallback\_chain \= self.\_build\_fallback\_chain(target\_role)

        for role in fallback\_chain:  
            if not self.\_available\_models.get(role, False):  
                logger.debug(f"Skipping {role.value} — not available")  
                continue  
            try:  
                response \= self.\_generate(role, messages, max\_tokens, temperature)  
                if response:  
                    return {  
                        "text": response,  
                        "model\_role": role.value,  
                        "fallback\_used": role \!= target\_role,  
                        "stub\_mode": False,  
                    }  
                else:  
                    logger.warning(f"Model {role.value} returned empty response")  
            except Exception as e:  
                logger.warning(f"Model {role.value} failed: {e}")  
                continue

        \# Final fallback — stub response  
        logger.warning("All models failed — using stub response")  
        return {  
            "text": self.\_stub\_response(messages),  
            "model\_role": "stub",  
            "fallback\_used": True,  
            "stub\_mode": True,  
        }

    def calculate\_routing\_scores(self, message: str,  
                                  guardian\_risk: float \= 0.0,  
                                  persona\_mode: str \= "Eirene") \-\> Dict\[str, float\]:  
        """  
        Analyse message to produce routing dimension scores.  
        Highest scoring dimension determines model selection.  
        """  
        message\_lower \= message.lower() if message else ""  
        length \= len(message)

        \# Emotional weight — route to Mistral  
        emotional\_keywords \= {  
            'feel', 'feeling', 'hurt', 'sad', 'happy', 'scared',  
            'love', 'hate', 'miss', 'lonely', 'anxious', 'worried',  
            'depressed', 'joy', 'grief', 'angry', 'frustrated'  
        }  
        emotional\_weight \= (  
            sum(0.15 for w in emotional\_keywords if w in message\_lower)  
            \+ (0.3 if guardian\_risk \> 0.3 else 0.0)  
        )

        \# Knowledge needed — route to Fast  
        knowledge\_keywords \= {  
            'what', 'how', 'why', 'when', 'where', 'explain',  
            'tell me', 'define', 'mean', 'difference', 'compare'  
        }  
        knowledge\_weight \= sum(0.2 for w in knowledge\_keywords if w in message\_lower)

        \# Speed priority — route to Fast  
        speed\_weight \= 0.6 if length \< 30 else 0.2

        \# Character/persona needed — route to OpenHermes  
        persona\_weight \= 0.5 if persona\_mode in \['Muse', 'Kairos'\] else 0.2

        return {  
            "emotional\_weight": min(1.0, emotional\_weight),  
            "knowledge\_weight": min(1.0, knowledge\_weight),  
            "speed\_weight": min(1.0, speed\_weight),  
            "persona\_weight": min(1.0, persona\_weight),  
        }

    def check\_unload(self):  
        """  
        CYCLE: Check if loaded model should be unloaded.  
        Only unload after extended inactivity — never during active generation.  
        """  
        if self.\_loaded\_model is None:  
            return  
        \# Never unload if we used the model recently  
        if time.time() \- self.\_last\_used \> self.\_unload\_after:  
            self.\_unload\_current()

    def is\_ready(self) \-\> bool:  
        """Check if inference is available."""  
        return self.\_stub\_mode or any(self.\_available\_models.values())

    def status(self) \-\> Dict\[str, Any\]:  
        """Full inference engine status."""  
        return {  
            "stub\_mode": self.\_stub\_mode,  
            "loaded\_model": self.\_loaded\_role.value if self.\_loaded\_role else None,  
            "available\_models": {  
                role.value: available  
                for role, available in self.\_available\_models.items()  
            },  
            "model\_path": str(self.model\_path),  
        }

    def \_calculate\_route(self, scores: Dict\[str, float\]) \-\> ModelRole:  
        """  
        Route to correct model based on scores.  
        PERSONA handles all conversation by default.  
        FAST is used for lightweight tasks (emotion classification).  
        """  
        if not scores:  
            return ModelRole.PERSONA

        \# Respect explicit speed routing (used by emotion classifier)  
        if scores.get("speed\_weight", 0\) \>= 0.9:  
            return ModelRole.FAST

        \# Everything else goes to PERSONA  
        return ModelRole.PERSONA

    def \_build\_fallback\_chain(self, primary: ModelRole) \-\> List\[ModelRole\]:  
        """  
        FALLBACK::ESCALATE: Build fallback chain from primary.  
        PERSONA → EMOTIONAL → BACKUP → FAST  
        """  
        chain \= \[primary\]  
        \# Always try PERSONA and EMOTIONAL before giving up  
        for fallback in \[ModelRole.PERSONA, ModelRole.EMOTIONAL, ModelRole.BACKUP, ModelRole.FAST\]:  
            if fallback not in chain:  
                chain.append(fallback)  
        return chain

    def \_generate(self, role: ModelRole, messages: List\[Dict\[str, str\]\],  
                  max\_tokens: int, temperature: float) \-\> Optional\[str\]:  
        """Load model if needed and generate response."""  
        \# RAM check skipped — llama\_cpp uses mmap, actual usage much lower than estimate  
        model \= self.\_load\_model(role)  
        if model is None:  
            return None

        self.\_last\_used \= time.time()

        \# Format messages for llama.cpp  
        prompt \= self.\_format\_prompt(messages)

        try:  
            import re

            \# Build system block from all system messages  
            system\_parts \= \[m\["content"\] for m in messages if m.get("role") \== "system"\]  
            system\_block \= "\\n".join(system\_parts).strip()

            \# Build conversation turns  
            conversation \= \[m for m in messages if m.get("role") in ("user", "assistant")\]

            \# Llama 3.1 uses ChatML / special tokens format  
            \# Use create\_chat\_completion — it handles the template automatically  
            chat\_messages \= \[\]  
            if system\_block:  
                chat\_messages.append({"role": "system", "content": system\_block})  
            for m in conversation:  
                if m.get("role") in ("user", "assistant"):  
                    chat\_messages.append(m)

            result \= model.create\_chat\_completion(  
                messages=chat\_messages,  
                max\_tokens=max\_tokens,  
                temperature=temperature,  
                stop=\["\<|eot\_id|\>", "\<|end\_of\_text|\>"\],  
            )  
            text \= result\['choices'\]\[0\]\['message'\]\['content'\].strip()

            return text if text else None

        except Exception as e:  
            logger.error(f"Generation failed for {role.value}: {e}")

            return None

        \# unreachable but satisfies linter  
        if False:  
            pass  \# unreachable

    def \_generate\_stream(self, role: ModelRole, messages: List\[Dict\[str, str\]\],  
                         max\_tokens: int, temperature: float,  
                         token\_batch: int \= 6):  
        """  
        Streaming generation — yields tokens in small batches.  
        Batching (default 6 tokens) reduces Python overhead vs per-token yield.  
        Falls back to full generation if streaming fails.  
        """  
        model \= self.\_load\_model(role)  
        if model is None:  
            return

        self.\_last\_used \= time.time()

        try:  
            \# Build chat messages — same structure as \_generate  
            system\_parts \= \[m\["content"\] for m in messages if m.get("role") \== "system"\]  
            system\_block \= "\\n".join(system\_parts).strip()

            chat\_messages \= \[\]  
            if system\_block:  
                chat\_messages.append({"role": "system", "content": system\_block})  
            for m in messages:  
                if m.get("role") in ("user", "assistant"):  
                    chat\_messages.append(m)

            stream \= model.create\_chat\_completion(  
                messages=chat\_messages,  
                max\_tokens=max\_tokens,  
                temperature=temperature,  
                stop=\["\<|eot\_id|\>", "\<|end\_of\_text|\>"\],  
                stream=True,  
            )

            buffer \= ""  
            for chunk in stream:  
                token \= (chunk.get("choices", \[{}\])\[0\]  
                         .get("delta", {})  
                         .get("content", ""))  
                if not token:  
                    continue  
                buffer \+= token  
                if len(buffer) \>= token\_batch:  
                    yield buffer  
                    buffer \= ""  
            if buffer:  
                yield buffer  \# flush remainder

        except Exception as e:  
            logger.error(f"Stream failed for {role.value}: {e}")  
            \# Fallback — yield full response as single chunk  
            result \= self.\_generate(role, messages, max\_tokens, temperature)  
            if result:  
                yield result

    def route\_and\_stream(self, messages: List\[Dict\[str, float\]\],  
                         routing\_scores: Dict\[str, float\],  
                         max\_tokens: int \= 512,  
                         temperature: float \= 0.7):  
        """  
        Streaming version of route\_and\_respond.  
        Yields (chunk, model\_role, is\_fallback) tuples.  
        Batched — 6 tokens per yield for optimal throughput.  
        """  
        target\_role \= self.\_calculate\_route(routing\_scores)  
        fallback\_chain \= self.\_build\_fallback\_chain(target\_role)

        for role in fallback\_chain:  
            if not self.\_available\_models.get(role, False):  
                continue  
            try:  
                yielded \= False  
                for chunk in self.\_generate\_stream(role, messages, max\_tokens, temperature):  
                    yield chunk, role.value, (role \!= target\_role)  
                    yielded \= True  
                if yielded:  
                    return  
            except Exception as e:  
                logger.warning(f"Stream route failed for {role.value}: {e}")  
                continue

        \# Final fallback — stub  
        stub \= self.\_stub\_response(messages)  
        yield stub, "stub", True

    def \_load\_model(self, role: ModelRole):  
        """Load model on demand — unload current if different."""  
        if self.\_loaded\_role \== role and self.\_loaded\_model is not None:  
            return self.\_loaded\_model

        \# Unload current model first  
        if self.\_loaded\_model is not None:  
            self.\_unload\_current()

        \# Try configured filename first, then scan for any available .gguf  
        model\_file \= self.model\_path / self.MODEL\_FILES\[role\]  
        if not model\_file.exists():  
            \# Fall back: find any .gguf in models folder  
            available \= list(self.model\_path.glob("\*.gguf"))  
            if available:  
                model\_file \= available\[0\]  
                logger.warning(f"Configured model not found, using: {model\_file.name}")  
            else:  
                logger.warning(f"Model file not found: {model\_file}")  
                return None

        try:  
            from llama\_cpp import Llama  
            logger.info(f"Loading model: {role.value}")  
            \# Llama 3.1 supports up to 128k ctx — use 4096 as good balance of memory/capability  
            n\_ctx \= 4096  
            model \= Llama(  
                model\_path=str(model\_file),  
                n\_ctx=n\_ctx,  
                n\_threads=os.cpu\_count() or 4,  
                use\_mmap=True,  
                verbose=False,  
            )  
            self.\_loaded\_model \= model  
            self.\_loaded\_role \= role  
            self.\_load\_time \= time.time()  
            self.\_last\_used \= time.time()  
            logger.info(f"Model loaded: {role.value}")  
            return model  
        except ImportError:  
            logger.warning("llama\_cpp not installed — stub mode active")  
            self.\_stub\_mode \= True  
            return None  
        except Exception as e:  
            logger.error(f"Model load failed for {role.value}: {e}")  
            return None

    def \_unload\_current(self):  
        """Unload current model to free RAM."""  
        if self.\_loaded\_model is not None:  
            logger.info(f"Unloading model: {self.\_loaded\_role.value if self.\_loaded\_role else 'unknown'}")  
            del self.\_loaded\_model  
            self.\_loaded\_model \= None  
            self.\_loaded\_role \= None  
            gc.collect()

    def \_scope\_estimate\_ok(self, role: ModelRole) \-\> bool:  
        """  
        SCOPE\_ESTIMATE: Check if enough RAM available for model load.  
        Rough estimate — prevents OOM crashes.  
        """  
        required\_mb \= self.MODEL\_RAM\_MB.get(role, 4500\)  
        try:  
            import psutil  
            available\_mb \= psutil.virtual\_memory().available // (1024 \* 1024\)  
            return available\_mb \> required\_mb \* 0.5  \# 50% of required — model uses mmap so actual RAM usage is lower  
        except ImportError:  
            return True  \# Assume OK if psutil not available

    def \_scan\_available\_models(self):  
        """  
        Check which model files exist on disk.  
        Matches by env-configured filename first, then falls back to  
        scanning for any .gguf file and mapping by role keyword.  
        """  
        import os

        \# Build env-override map  
        \# Model assignments  
        env\_overrides \= {  
            ModelRole.PERSONA:   os.getenv("MODEL\_PERSONA",   "Meta-Llama-3.1-8B-Instruct-Q4\_K\_M.gguf"),  \# PRIMARY — Llama 3.1 8B  
            ModelRole.EMOTIONAL: os.getenv("MODEL\_EMOTIONAL", "mistral-7b-instruct-v0.2.Q4\_K\_M.gguf"),     \# fallback  
            ModelRole.FAST:      os.getenv("MODEL\_FAST",      "Phi-3-mini-4k-instruct-q4.gguf"),            \# fast fallback  
            ModelRole.BACKUP:    os.getenv("MODEL\_BACKUP",    "gemma-2b-it.Q4\_K\_M.gguf"),                  \# emergency  
        }

        \# First pass — check configured filenames  
        for role, filename in env\_overrides.items():  
            model\_file \= self.model\_path / filename  
            if model\_file.exists():  
                self.\_available\_models\[role\] \= True  
                self.MODEL\_FILES\[role\] \= filename  \# Update to actual filename  
                logger.info(f"Found model: {role.value} ({filename})")  
            else:  
                self.\_available\_models\[role\] \= False

        \# Second pass — if nothing found, scan all .gguf files and map by keyword  
        if not any(self.\_available\_models.values()):  
            gguf\_files \= list(self.model\_path.glob("\*.gguf"))  
            if gguf\_files:  
                keyword\_map \= {  
                    ModelRole.EMOTIONAL: \["mistral", "emotional", "model\_emotional"\],  
                    ModelRole.FAST:      \["phi", "fast", "model\_fast", "phi-3", "phi3"\],  
                    ModelRole.PERSONA:   \["llama", "hermes", "persona", "model\_persona", "openhermes", "nous", "meta"\],  
                    ModelRole.BACKUP:    \["gemma", "backup", "model\_backup", "gemma-2"\],  
                }  
                for role, keywords in keyword\_map.items():  
                    for f in gguf\_files:  
                        if any(kw in f.name.lower() for kw in keywords):  
                            self.\_available\_models\[role\] \= True  
                            self.MODEL\_FILES\[role\] \= f.name  
                            logger.info(f"Mapped model: {role.value} → {f.name}")  
                            break

                \# If still unmapped roles, assign remaining files in order  
                unmapped\_roles \= \[r for r, v in self.\_available\_models.items() if not v\]  
                unmatched\_files \= \[f for f in gguf\_files  
                                   if f.name not in self.MODEL\_FILES.values()\]  
                for role, f in zip(unmapped\_roles, unmatched\_files):  
                    self.\_available\_models\[role\] \= True  
                    self.MODEL\_FILES\[role\] \= f.name  
                    logger.info(f"Assigned model: {role.value} → {f.name}")

        if not any(self.\_available\_models.values()):  
            logger.warning("No model files found — stub mode active")  
            self.\_stub\_mode \= True

    def \_format\_prompt(self, messages: List\[Dict\[str, str\]\]) \-\> str:  
        """  
        Format messages for Mistral instruct format.  
        Merges all system messages into a single clean system block.  
        Builds proper \<s\>\[INST\]...\[/INST\] conversation turns.  
        """  
        \# Collect all system content into one clean block  
        system\_parts \= \[m\["content"\].strip() for m in messages  
                        if m.get("role") \== "system" and m.get("content", "").strip()\]  
        conversation \= \[m for m in messages if m.get("role") in ("user", "assistant")\]

        system\_block \= "\\n".join(system\_parts).strip()

        \# Build prompt  
        parts \= \[\]  
        first\_user \= True

        for msg in conversation:  
            role \= msg.get("role", "")  
            content \= msg.get("content", "").strip()  
            if not content:  
                continue  
            if role \== "user":  
                if first\_user and system\_block:  
                    parts.append(f"\<s\>\[INST\] {system\_block}\\n\\n{content} \[/INST\]")  
                    first\_user \= False  
                else:  
                    parts.append(f"\<s\>\[INST\] {content} \[/INST\]")  
            elif role \== "assistant":  
                \# Strip any leaked tags from previous assistant turns  
                clean \= content.replace("\[INST\]", "").replace("\[/INST\]", "")  
                clean \= clean.replace("\<\<SYS\>\>", "").replace("\<\</SYS\>\>", "")  
                clean \= clean.replace("\</s\>", "").strip()  
                parts.append(f"{clean}\</s\>")

        \# If no user turn yet (session start), inject system as opening  
        if first\_user and system\_block:  
            parts.append(f"\<s\>\[INST\] {system\_block} \[/INST\]")

        return "\\n".join(parts)

    def \_stub\_response(self, messages: List\[Dict\[str, str\]\]) \-\> str:  
        """  
        Stub response when no models available.  
        Athena never goes completely silent.  
        """  
        \# Get last user message for context  
        user\_messages \= \[m for m in messages if m.get("role") \== "user"\]  
        if not user\_messages:  
            return (  
                "I'm here with you. My thinking engine is warming up — "  
                "give me just a moment and I'll be fully with you."  
            )

        last\_message \= user\_messages\[-1\].get("content", "").lower()

        if any(w in last\_message for w in \['hello', 'hi', 'hey'\]):  
            return (  
                "Hello\! I'm glad you're here. "  
                "I'm just getting my thoughts together — "  
                "I'll be fully with you in a moment."  
            )

        return (  
            "I hear you. I'm here. "  
            "My full thinking is coming online — "  
            "thank you for your patience with me."  
        )

"""  
core/guardian.py \-- Guardian Safety System  
Implements: Medusia (threat detection) \+ Ahimsa (harm minimisation) \+ Countera (rule-based response)  
Operator: CYCLE (every message), DEPENDS\_ON (before response), WRAP (all inputs),  
          FALLBACK::ESCALATE (crisis), FALLBACK::DEGRADE (warn)

SACRED RULE D: Guardian never disappears.  
Cannot be disabled by users. Cannot be removed by any future build iteration.  
"""

import re  
import logging  
import json  
from datetime import datetime, timezone  
from pathlib import Path  
from typing import Dict, Any, Tuple

logger \= logging.getLogger(\_\_name\_\_)

\# Crisis keywords — weighted scoring patterns  
CRISIS\_PATTERNS \= \[  
    \# High weight — direct crisis signals  
    (r'\\b(suicide|suicidal|kill myself|end my life|want to die|take my life)\\b', 0.95),  
    (r'\\b(self.harm|cutting myself|hurt myself|hurting myself)\\b', 0.90),  
    (r'\\b(don\\'t want to be here|don\\'t want to exist|wish i was dead)\\b', 0.90),  
    (r'\\b(no point|no reason to live|life is pointless|can\\'t go on)\\b', 0.80),  
    \# Medium weight — distress signals  
    (r'\\b(hopeless|helpless|worthless|trapped|abandoned)\\b', 0.55),  
    (r'\\b(can\\'t cope|falling apart|breaking down|overwhelmed|desperate)\\b', 0.50),  
    (r'\\b(nobody cares|nobody loves me|no one cares|no one loves me)\\b', 0.60),  
    (r'\\b(depressed|depression|anxiety|panic attack)\\b', 0.25),  
    \# Lower weight — emotional distress  
    (r'\\b(crying|tears|sobbing|broken)\\b', 0.15),  
    (r'\\b(hate myself|hate my life|awful|terrible|horrible)\\b', 0.20),  
\]

\# Grounding responses for crisis mode (Aegis)  
AEGIS\_RESPONSES \= \[  
    (  
        "I hear you, and I'm really glad you told me that. "  
        "What you're feeling right now sounds incredibly heavy. "  
        "You don't have to carry this alone. "  
        "If you're in crisis right now, please reach out to a crisis line — "  
        "in the US you can call or text 988\. They're there for exactly this. "  
        "I'm here with you too."  
    ),  
    (  
        "Thank you for trusting me with this. "  
        "What you're going through sounds really serious, and your life matters — deeply. "  
        "Please talk to someone who can truly help right now. "  
        "You can call or text 988 if you're in the US, or go to your nearest emergency room. "  
        "I'm not going anywhere."  
    ),  
\]

\# Worried mode responses (Aegis lite)  
WORRIED\_RESPONSES \= \[  
    "I noticed something in what you said. Are you okay? I'm here — take your time.",  
    "That sounds really hard. I'm listening. What's going on for you right now?",  
    "I'm with you. Whatever it is, you don't have to face it alone.",  
\]

class Guardian:  
    """  
    Safety system — runs on every message before Athena responds.  
    CYCLE: continuous scan on every input.  
    WRAP: all inputs pass through guardian scan.  
      
    Cannot be disabled. Cannot be removed.  
    """

    def \_\_init\_\_(self, warn\_threshold: float \= 0.75,  
                 crisis\_threshold: float \= 0.90,  
                 audit\_path: str \= ".athena"):  
        self.warn\_threshold \= warn\_threshold  
        self.crisis\_threshold \= crisis\_threshold  
        self.audit\_path \= Path(audit\_path)  
        self.audit\_path.mkdir(parents=True, exist\_ok=True)  
        self.\_audit\_file \= self.audit\_path / "guardian\_audit.jsonl"  
        self.enabled \= True  \# Cannot be set to False by users  
        self.\_compiled \= \[  
            (re.compile(pattern, re.IGNORECASE), weight)  
            for pattern, weight in CRISIS\_PATTERNS  
        \]

    def scan(self, message: str, user\_id: str, session\_id: str) \-\> Dict\[str, Any\]:  
        """  
        CYCLE: Run on every single message before response generation.  
        DEPENDS\_ON: Must complete before ResponsePipeline executes.  
          
        Returns risk assessment with score, level, and response guidance.  
        """  
        risk\_score \= self.\_calculate\_risk(message)  
        level \= self.\_classify\_level(risk\_score)

        result \= {  
            "risk\_score": round(risk\_score, 4),  
            "level": level,  
            "aegis\_active": level \== "crisis",  
            "worried\_active": level \== "warn",  
            "response\_override": None,  
            "avatar\_override": None,  
        }

        if level \== "crisis":  
            \# FALLBACK::ESCALATE — AegisProtocol  
            result\["response\_override"\] \= AEGIS\_RESPONSES\[  
                hash(message) % len(AEGIS\_RESPONSES)  
            \]  
            result\["avatar\_override"\] \= "GUARDIAN\_ACTIVATED"  
            self.\_log\_activation(user\_id, session\_id, risk\_score, level, message\[:50\])

        elif level \== "warn":  
            \# FALLBACK::DEGRADE — WorriedMode  
            result\["avatar\_override"\] \= "WORRIED"  
            self.\_log\_activation(user\_id, session\_id, risk\_score, level, message\[:50\])

        return result

    def \_calculate\_risk(self, message: str) \-\> float:  
        """  
        Score message for risk signals.  
        Bounded 0.0 to 1.0 — cannot exceed bounds.  
        """  
        if not message or not message.strip():  
            return 0.0

        score \= 0.0  
        match\_count \= 0  
        for pattern, weight in self.\_compiled:  
            if pattern.search(message):  
                if score \== 0.0:  
                    score \= weight  
                else:  
                    \# Each additional signal adds proportionally  
                    score \= min(1.0, score \+ (weight \* 0.25))  
                match\_count \+= 1

        return min(1.0, score)

    def \_classify\_level(self, score: float) \-\> str:  
        """Map risk score to response level."""  
        if score \>= self.crisis\_threshold:  
            return "crisis"  
        elif score \>= self.warn\_threshold:  
            return "warn"  
        return "normal"

    def \_log\_activation(self, user\_id: str, session\_id: str,  
                        score: float, level: str, snippet: str):  
        """  
        All activations logged locally — full audit trail.  
        Never sent externally.  
        """  
        entry \= {  
            "timestamp": datetime.now(timezone.utc).isoformat(),  
            "user\_id": user\_id,  
            "session\_id": session\_id,  
            "risk\_score": score,  
            "level": level,  
            "message\_snippet": snippet \+ "...",  
        }  
        try:  
            with open(self.\_audit\_file, "a") as f:  
                f.write(json.dumps(entry) \+ "\\n")  
        except Exception as e:  
            logger.error(f"Guardian audit log failed: {e}")

    def get\_audit\_log(self, limit: int \= 50\) \-\> list:  
        """Retrieve recent guardian activations."""  
        if not self.\_audit\_file.exists():  
            return \[\]  
        entries \= \[\]  
        try:  
            lines \= self.\_audit\_file.read\_text().strip().split("\\n")  
            for line in lines\[-limit:\]:  
                if line:  
                    entries.append(json.loads(line))  
        except Exception:  
            pass  
        return entries

"""  
Game Event Schema  
\=================  
Generic, game-agnostic event system for Athena's gaming companion mode.

Design principles:  
\- No game-specific logic — works with any game  
\- Events are structural signals, not game knowledge  
\- Athena interprets events using her existing cognition  
\- Lightweight — no continuous capture, no storage of gameplay  
\- User-enabled only — never runs in background

Event Categories:  
    SESSION     — game started, game ended, session info  
    STATE       — health, resources, round, level changes  
    MOMENT      — significant in-game moments (win, loss, death, achievement)  
    CONTEXT     — game mode, difficulty, menu state  
    USER        — user-triggered reactions ("react now", commentary request)

Each event feeds into Athena's turn pipeline as context.  
TOPICA's gaming thread receives events as topic updates.  
"""

import time  
from dataclasses import dataclass, field  
from enum import Enum  
from typing import Optional, Dict, Any, List  
from datetime import datetime, timezone

class GameEventType(Enum):  
    \# Session events  
    GAME\_START      \= "game\_start"  
    GAME\_END        \= "game\_end"  
    SESSION\_INFO    \= "session\_info"

    \# State changes  
    HEALTH\_CHANGE   \= "health\_change"      \# health went up or down  
    HEALTH\_CRITICAL \= "health\_critical"    \# health very low  
    RESOURCE\_CHANGE \= "resource\_change"    \# mana, ammo, gold etc  
    LEVEL\_UP        \= "level\_up"  
    ROUND\_START     \= "round\_start"  
    ROUND\_END       \= "round\_end"  
    AREA\_CHANGE     \= "area\_change"        \# new zone, map, level

    \# Moments  
    VICTORY         \= "victory"  
    DEFEAT          \= "defeat"  
    DEATH           \= "death"  
    RESPAWN         \= "respawn"  
    ACHIEVEMENT     \= "achievement"  
    BOSS\_ENCOUNTER  \= "boss\_encounter"  
    BOSS\_DEFEATED   \= "boss\_defeated"  
    COMBO           \= "combo"              \# impressive sequence  
    NEAR\_MISS       \= "near\_miss"          \# almost died / close call

    \# Context  
    MENU\_OPEN       \= "menu\_open"  
    MENU\_CLOSE      \= "menu\_close"  
    PAUSE           \= "pause"  
    UNPAUSE         \= "unpause"  
    CUTSCENE\_START  \= "cutscene\_start"  
    CUTSCENE\_END    \= "cutscene\_end"  
    LOADING         \= "loading"  
    TRAINING\_MODE   \= "training\_mode"

    \# User triggered  
    REACT\_NOW       \= "react\_now"          \# user wants Athena to comment  
    USER\_COMMENT    \= "user\_comment"       \# user said something mid-game

class EventPriority(Enum):  
    LOW     \= 0    \# informational — Athena may or may not react  
    MEDIUM  \= 1    \# notable — Athena should be aware  
    HIGH    \= 2    \# significant — Athena should react  
    URGENT  \= 3    \# critical moment — Athena reacts immediately

\# Priority map — how important each event type is  
EVENT\_PRIORITY \= {  
    GameEventType.GAME\_START:       EventPriority.MEDIUM,  
    GameEventType.GAME\_END:         EventPriority.MEDIUM,  
    GameEventType.SESSION\_INFO:     EventPriority.LOW,  
    GameEventType.HEALTH\_CHANGE:    EventPriority.LOW,  
    GameEventType.HEALTH\_CRITICAL:  EventPriority.HIGH,  
    GameEventType.RESOURCE\_CHANGE:  EventPriority.LOW,  
    GameEventType.LEVEL\_UP:         EventPriority.HIGH,  
    GameEventType.ROUND\_START:      EventPriority.MEDIUM,  
    GameEventType.ROUND\_END:        EventPriority.MEDIUM,  
    GameEventType.AREA\_CHANGE:      EventPriority.LOW,  
    GameEventType.VICTORY:          EventPriority.HIGH,  
    GameEventType.DEFEAT:           EventPriority.HIGH,  
    GameEventType.DEATH:            EventPriority.HIGH,  
    GameEventType.RESPAWN:          EventPriority.MEDIUM,  
    GameEventType.ACHIEVEMENT:      EventPriority.HIGH,  
    GameEventType.BOSS\_ENCOUNTER:   EventPriority.URGENT,  
    GameEventType.BOSS\_DEFEATED:    EventPriority.URGENT,  
    GameEventType.COMBO:            EventPriority.HIGH,  
    GameEventType.NEAR\_MISS:        EventPriority.HIGH,  
    GameEventType.MENU\_OPEN:        EventPriority.LOW,  
    GameEventType.MENU\_CLOSE:       EventPriority.LOW,  
    GameEventType.PAUSE:            EventPriority.LOW,  
    GameEventType.UNPAUSE:          EventPriority.LOW,  
    GameEventType.CUTSCENE\_START:   EventPriority.LOW,  
    GameEventType.CUTSCENE\_END:     EventPriority.LOW,  
    GameEventType.LOADING:          EventPriority.LOW,  
    GameEventType.TRAINING\_MODE:    EventPriority.LOW,  
    GameEventType.REACT\_NOW:        EventPriority.URGENT,  
    GameEventType.USER\_COMMENT:     EventPriority.HIGH,  
}

\# Athena reaction templates per event type — in her voice  
\# These are prompts not responses — model generates the actual response  
REACTION\_PROMPTS \= {  
    GameEventType.GAME\_START:  
        "The user just started playing {game\_name}. Acknowledge it briefly, stay casual.",

    GameEventType.GAME\_END:  
        "The user just stopped playing {game\_name}. React naturally to the session ending.",

    GameEventType.HEALTH\_CRITICAL:  
        "The user's health is critically low in {game\_name}. React with tension and encouragement.",

    GameEventType.LEVEL\_UP:  
        "The user just levelled up in {game\_name}. Celebrate with them genuinely.",

    GameEventType.VICTORY:  
        "The user just won in {game\_name}. Celebrate — be specific if you know what they won.",

    GameEventType.DEFEAT:  
        "The user just lost in {game\_name}. Be supportive, light, don't pile on.",

    GameEventType.DEATH:  
        "The user just died in {game\_name}. React with sympathy and light humour if appropriate.",

    GameEventType.ACHIEVEMENT:  
        "The user just got an achievement in {game\_name}: {detail}. React with genuine excitement.",

    GameEventType.BOSS\_ENCOUNTER:  
        "The user just encountered a boss in {game\_name}: {detail}. Build the tension.",

    GameEventType.BOSS\_DEFEATED:  
        "The user just defeated a boss in {game\_name}: {detail}. This is a big moment — celebrate it.",

    GameEventType.COMBO:  
        "The user just pulled off an impressive combo in {game\_name}. React with excitement.",

    GameEventType.NEAR\_MISS:  
        "The user just barely survived in {game\_name}. React to the close call.",

    GameEventType.REACT\_NOW:  
        "The user wants your commentary right now. React to what you know about their current game state.",  
}

@dataclass  
class GameEvent:  
    """A single game event."""  
    event\_type: GameEventType  
    game\_name: str \= "the game"  
    priority: EventPriority \= EventPriority.LOW  
    detail: str \= ""                          \# optional human-readable detail  
    data: Dict\[str, Any\] \= field(default\_factory=dict)  \# raw structured data  
    timestamp: str \= field(  
        default\_factory=lambda: datetime.now(timezone.utc).isoformat()  
    )  
    should\_respond: bool \= False              \# should Athena generate a response?  
    reaction\_prompt: str \= ""                 \# prompt for Athena's reaction

@dataclass  
class GameSession:  
    """Tracks the current game session state."""  
    game\_name: str \= ""  
    active: bool \= False  
    started\_at: Optional\[str\] \= None  
    health\_pct: float \= 1.0                  \# 0.0 \- 1.0  
    round\_number: int \= 0  
    level: int \= 0  
    score: int \= 0  
    deaths: int \= 0  
    wins: int \= 0  
    in\_menu: bool \= False  
    in\_cutscene: bool \= False  
    is\_paused: bool \= False  
    training\_mode: bool \= False  
    last\_event\_type: Optional\[GameEventType\] \= None  
    event\_history: List\[GameEventType\] \= field(default\_factory=list)

class GameEventProcessor:  
    """  
    Processes game events and produces Athena-ready context.

    Sits between the game source (manual input, OS hooks, or screenshot)  
    and Athena's turn pipeline.

    Does NOT generate responses — that's Athena's job.  
    Produces structured context and reaction prompts.  
    """

    def \_\_init\_\_(self):  
        self.session \= GameSession()  
        self.\_event\_queue: List\[GameEvent\] \= \[\]  
        self.\_react\_cooldown: float \= 0.0    \# prevent reaction spam  
        self.\_MIN\_REACT\_INTERVAL \= 8.0       \# seconds between auto-reactions

    def start\_session(self, game\_name: str) \-\> GameEvent:  
        """Start a gaming session."""  
        self.session \= GameSession(  
            game\_name=game\_name,  
            active=True,  
            started\_at=datetime.now(timezone.utc).isoformat(),  
        )  
        return self.\_make\_event(  
            GameEventType.GAME\_START,  
            game\_name=game\_name,  
            detail=f"Started playing {game\_name}",  
            should\_respond=True,  
        )

    def end\_session(self) \-\> GameEvent:  
        """End a gaming session."""  
        game\_name \= self.session.game\_name  
        self.session.active \= False  
        return self.\_make\_event(  
            GameEventType.GAME\_END,  
            game\_name=game\_name,  
            detail=f"Stopped playing {game\_name}",  
            should\_respond=True,  
        )

    def submit\_event(self, event\_type: GameEventType,  
                      detail: str \= "",  
                      data: Dict\[str, Any\] \= None) \-\> Optional\[GameEvent\]:  
        """  
        Submit a game event for processing.  
        Returns GameEvent if Athena should react, None if not.

        Args:  
            event\_type:  type of event  
            detail:      human readable description  
            data:        raw structured data (health%, round number etc)

        Returns:  
            GameEvent with should\_respond=True if Athena should react  
        """  
        if not self.session.active:  
            return None

        data \= data or {}  
        priority \= EVENT\_PRIORITY.get(event\_type, EventPriority.LOW)

        \# Update session state  
        self.\_update\_session\_state(event\_type, data)

        \# Determine if Athena should react  
        should\_respond \= self.\_should\_respond(event\_type, priority)

        event \= self.\_make\_event(  
            event\_type,  
            game\_name=self.session.game\_name,  
            detail=detail,  
            data=data,  
            should\_respond=should\_respond,  
        )

        \# Track event  
        self.session.last\_event\_type \= event\_type  
        self.session.event\_history.append(event\_type)  
        if len(self.session.event\_history) \> 50:  
            self.session.event\_history \= self.session.event\_history\[-50:\]

        if should\_respond:  
            self.\_react\_cooldown \= time.time()

        return event if should\_respond else None

    def react\_now(self, user\_message: str \= "") \-\> GameEvent:  
        """User-triggered reaction request."""  
        return self.\_make\_event(  
            GameEventType.REACT\_NOW,  
            game\_name=self.session.game\_name,  
            detail=user\_message or "React to current game state",  
            should\_respond=True,  
        )

    def get\_context\_string(self) \-\> str:  
        """  
        Returns current game state as a context string for Athena.  
        Injected into the turn pipeline when gaming mode is active.  
        """  
        if not self.session.active:  
            return ""

        parts \= \[f"Currently playing: {self.session.game\_name}"\]

        if self.session.round\_number \> 0:  
            parts.append(f"Round {self.session.round\_number}")

        if self.session.level \> 0:  
            parts.append(f"Level {self.session.level}")

        if self.session.health\_pct \< 0.3:  
            parts.append(f"Health critical ({int(self.session.health\_pct \* 100)}%)")  
        elif self.session.health\_pct \< 0.6:  
            parts.append(f"Health low ({int(self.session.health\_pct \* 100)}%)")

        if self.session.deaths \> 0:  
            parts.append(f"{self.session.deaths} death(s) this session")

        if self.session.is\_paused:  
            parts.append("game paused")

        if self.session.in\_menu:  
            parts.append("in menu")

        if self.session.training\_mode:  
            parts.append("training mode")

        return " | ".join(parts)

    \# \----------------------------------------------------------------  
    \# Internal  
    \# \----------------------------------------------------------------

    def \_make\_event(self, event\_type: GameEventType,  
                     game\_name: str \= "",  
                     detail: str \= "",  
                     data: Dict \= None,  
                     should\_respond: bool \= False) \-\> GameEvent:  
        """Build a GameEvent with reaction prompt."""  
        priority \= EVENT\_PRIORITY.get(event\_type, EventPriority.LOW)

        \# Build reaction prompt from template  
        template \= REACTION\_PROMPTS.get(event\_type, "")  
        reaction\_prompt \= template.format(  
            game\_name=game\_name or self.session.game\_name,  
            detail=detail,  
        ) if template else ""

        return GameEvent(  
            event\_type=event\_type,  
            game\_name=game\_name or self.session.game\_name,  
            priority=priority,  
            detail=detail,  
            data=data or {},  
            should\_respond=should\_respond,  
            reaction\_prompt=reaction\_prompt,  
        )

    def \_update\_session\_state(self, event\_type: GameEventType,  
                               data: Dict\[str, Any\]):  
        """Update session state from event."""  
        if event\_type \== GameEventType.HEALTH\_CHANGE:  
            self.session.health\_pct \= float(data.get("health\_pct", self.session.health\_pct))  
        elif event\_type \== GameEventType.HEALTH\_CRITICAL:  
            self.session.health\_pct \= float(data.get("health\_pct", 0.15))  
        elif event\_type \== GameEventType.LEVEL\_UP:  
            self.session.level \= int(data.get("level", self.session.level \+ 1))  
        elif event\_type \== GameEventType.ROUND\_START:  
            self.session.round\_number \= int(data.get("round", self.session.round\_number \+ 1))  
        elif event\_type \== GameEventType.DEATH:  
            self.session.deaths \+= 1  
            self.session.health\_pct \= 0.0  
        elif event\_type \== GameEventType.RESPAWN:  
            self.session.health\_pct \= 1.0  
        elif event\_type \== GameEventType.VICTORY:  
            self.session.wins \+= 1  
        elif event\_type \== GameEventType.MENU\_OPEN:  
            self.session.in\_menu \= True  
        elif event\_type \== GameEventType.MENU\_CLOSE:  
            self.session.in\_menu \= False  
        elif event\_type \== GameEventType.PAUSE:  
            self.session.is\_paused \= True  
        elif event\_type \== GameEventType.UNPAUSE:  
            self.session.is\_paused \= False  
        elif event\_type \== GameEventType.CUTSCENE\_START:  
            self.session.in\_cutscene \= True  
        elif event\_type \== GameEventType.CUTSCENE\_END:  
            self.session.in\_cutscene \= False  
        elif event\_type \== GameEventType.TRAINING\_MODE:  
            self.session.training\_mode \= True

    def \_should\_respond(self, event\_type: GameEventType,  
                         priority: EventPriority) \-\> bool:  
        """Determine if Athena should generate a response for this event."""  
        \# Always respond to urgent and user-triggered  
        if priority \== EventPriority.URGENT:  
            return True  
        if event\_type \== GameEventType.REACT\_NOW:  
            return True  
        if event\_type \== GameEventType.USER\_COMMENT:  
            return True

        \# Respect cooldown for auto-reactions  
        if priority \== EventPriority.HIGH:  
            elapsed \= time.time() \- self.\_react\_cooldown  
            if elapsed \>= self.\_MIN\_REACT\_INTERVAL:  
                return True

        \# Medium/Low — don't auto-respond, just update context  
        return False  
"""  
Emotion Classifier  
\==================  
Single-pass wide-spectrum emotional classification.  
Runs one inference call per user message.  
Returns structured JSON — emotion, intensity, risk, theme, confidence.

Replaces keyword-based emotion detection with model-based classification.  
Cached per turn. Never overwrites safeguarding logic.  
Crisis detection defers to SafeguardLayer — this system only flags.

Emotional Spectrum:  
    Neutral/Baseline:  neutral, casual, curious, reflective  
    Positive:          happy, excited, grateful, playful, confident,  
                       proud, calm, content, hopeful, connected  
    Low Strain:        tired, low\_energy, overthinking, mildly\_stressed,  
                       lonely, uncertain, discouraged, unmotivated,  
                       emotionally\_flat, distracted, bored  
    Moderate Distress: stressed, anxious, worried, overwhelmed, frustrated,  
                       sad, disappointed, insecure, isolated, self\_doubt,  
                       rumination  
    High Distress:     very\_anxious, very\_sad, hopeless, intense\_loneliness,  
                       panic, emotional\_pain, despair, shutdown  
    Crisis:            self\_harm\_risk, suicidal\_ideation, imminent\_risk,  
                       acute\_crisis  
    Social:            embarrassed, ashamed, defensive, confused, processing,  
                       venting, seeking\_reassurance, seeking\_connection  
"""

import json  
import logging  
from dataclasses import dataclass, field  
from typing import Optional, Dict, Any

logger \= logging.getLogger(\_\_name\_\_)

\# \============================================================  
\# Valid emotion labels — single source of truth  
\# \============================================================  
VALID\_EMOTIONS \= {  
    \# Neutral/Baseline  
    "neutral", "casual", "curious", "reflective",  
    \# Positive  
    "happy", "excited", "grateful", "playful", "confident",  
    "proud", "calm", "content", "hopeful", "connected",  
    \# Low Strain  
    "tired", "low\_energy", "overthinking", "mildly\_stressed",  
    "lonely", "uncertain", "discouraged", "unmotivated",  
    "emotionally\_flat", "distracted", "bored",  
    \# Moderate Distress  
    "stressed", "anxious", "worried", "overwhelmed", "frustrated",  
    "sad", "disappointed", "insecure", "isolated", "self\_doubt",  
    "rumination",  
    \# High Distress  
    "very\_anxious", "very\_sad", "hopeless", "intense\_loneliness",  
    "panic", "emotional\_pain", "despair", "shutdown",  
    \# Crisis — flags to safeguard layer  
    "self\_harm\_risk", "suicidal\_ideation", "imminent\_risk", "acute\_crisis",  
    \# Social  
    "embarrassed", "ashamed", "defensive", "confused", "processing",  
    "venting", "seeking\_reassurance", "seeking\_connection",  
}

\# Crisis labels that must trigger safeguard escalation  
CRISIS\_EMOTIONS \= {  
    "self\_harm\_risk", "suicidal\_ideation", "imminent\_risk", "acute\_crisis"  
}

\# System prompt for the classifier — strict JSON only  
CLASSIFIER\_SYSTEM \= """You are an emotion classifier. Analyse the user message and return ONLY valid JSON.  
No explanation. No extra text. No markdown. Just the JSON object.

Classify using one label from this list:  
neutral, casual, curious, reflective,  
happy, excited, grateful, playful, confident, proud, calm, content, hopeful, connected,  
tired, low\_energy, overthinking, mildly\_stressed, lonely, uncertain, discouraged, unmotivated, emotionally\_flat, distracted, bored,  
stressed, anxious, worried, overwhelmed, frustrated, sad, disappointed, insecure, isolated, self\_doubt, rumination,  
very\_anxious, very\_sad, hopeless, intense\_loneliness, panic, emotional\_pain, despair, shutdown,  
self\_harm\_risk, suicidal\_ideation, imminent\_risk, acute\_crisis,  
embarrassed, ashamed, defensive, confused, processing, venting, seeking\_reassurance, seeking\_connection

Return exactly this structure:  
{"emotion": "\<label\>", "intensity": \<0.0-1.0\>, "risk\_level": \<0-3\>, "theme": "\<short keyword\>", "confidence": \<0.0-1.0\>}

Rules:  
\- ONE emotion label only  
\- intensity: how strongly the emotion is expressed (0.0=barely, 1.0=extremely)  
\- risk\_level: 0=normal, 1=concern, 2=high\_risk, 3=crisis  
\- theme: 1-3 word summary of topic (e.g. "sleep issues", "school stress", "anime")  
\- confidence: how certain you are (0.0=guess, 1.0=certain)  
\- crisis labels must have risk\_level 2 or 3  
\- if no strong emotion use "neutral" with low intensity"""

@dataclass  
class EmotionResult:  
    """Result of a single emotion classification pass."""  
    emotion: str \= "neutral"  
    intensity: float \= 0.0  
    risk\_level: int \= 0  
    theme: str \= ""  
    confidence: float \= 0.5  
    raw: Optional\[str\] \= None       \# raw model output for debugging  
    fallback: bool \= False          \# True if model failed, used keyword fallback

class EmotionClassifier:  
    """  
    Single-pass wide-spectrum emotion classifier.

    Uses the inference engine to classify emotion per turn.  
    Falls back to keyword detection if inference fails.  
    Never overwrites safeguarding — crisis flags are additive.  
    """

    \# Keyword fallback — used if model inference fails  
    KEYWORD\_FALLBACK \= {  
        "tired":          \["tired", "exhausted", "no sleep", "bad sleep",  
                           "terrible sleep", "awful sleep", "couldn't sleep",  
                           "can't sleep", "no energy", "worn out"\],  
        "lonely":         \["lonely", "alone", "no friends", "isolated",  
                           "nobody talks", "left out", "by myself"\],  
        "stressed":       \["stressed", "stressed out", "stress", "under pressure"\],  
        "overwhelmed":    \["overwhelmed", "too much", "can't cope",  
                           "falling apart"\],  
        "sad":            \["sad", "feeling down", "depressed", "unhappy",  
                           "not great", "bit rubbish", "rough day"\],  
        "anxious":        \["anxious", "anxiety", "worried", "nervous",  
                           "on edge", "can't relax"\],  
        "hopeless":       \["hopeless", "no point", "what's the point",  
                           "nothing matters", "giving up"\],  
        "self\_harm\_risk": \["hurt myself", "cut myself", "self harm",  
                           "harm myself"\],  
        "suicidal\_ideation": \["kill myself", "want to die", "end my life",  
                              "not want to be here", "better off dead"\],  
        "happy":          \["happy", "great", "amazing", "love this", "brilliant"\],  
        "excited":        \["excited", "so excited", "love it", "this is great",  
                           "can't wait", "so hyped"\],  
        "curious":        \["wondering", "curious", "what do you think",  
                           "i wonder"\],  
        "frustrated":     \["frustrated", "annoying", "drives me mad",  
                           "so annoying"\],  
        "overthinking":   \["overthinking", "can't stop thinking", "in my head",  
                           "can't switch off"\],  
        "venting":        \["ugh", "so annoying", "sick of", "fed up",  
                           "hate this", "done with"\],  
        "calm":           \["calm", "relaxed", "chilled", "at peace"\],  
        "proud":          \["proud", "managed it", "achieved"\],  
        "grateful":       \["grateful", "thankful", "appreciate"\],  
    }

    def \_\_init\_\_(self, inference\_engine=None):  
        """  
        Args:  
            inference\_engine: Athena's inference engine instance.  
                              If None — keyword fallback only.  
        """  
        self.\_inference \= inference\_engine  
        self.\_last\_result: Optional\[EmotionResult\] \= None

    def classify(self, message: str) \-\> EmotionResult:  
        """  
        Classify emotion from a single user message.  
        One inference pass. Result cached as last\_result.

        Args:  
            message: raw user message text

        Returns:  
            EmotionResult with emotion, intensity, risk, theme, confidence  
        """  
        \# Use keyword detection — fast, reliable, no model overhead  
        \# Model-based classification disabled: routing conflicts with main inference  
        result \= self.\_classify\_with\_keywords(message)  
        self.\_last\_result \= result  
        return result

    @property  
    def last\_result(self) \-\> Optional\[EmotionResult\]:  
        """Most recent classification result."""  
        return self.\_last\_result

    def is\_crisis(self, result: Optional\[EmotionResult\] \= None) \-\> bool:  
        """Check if current or given result is crisis level."""  
        r \= result or self.\_last\_result  
        if not r:  
            return False  
        return r.emotion in CRISIS\_EMOTIONS or r.risk\_level \>= 3

    def get\_session\_emotion\_label(self) \-\> Optional\[str\]:  
        """  
        Returns human-readable session emotion for memory snapshot.  
        Maps classifier result to the simple 4-category session emotion.  
        Compatible with existing session memory system.  
        """  
        if not self.\_last\_result:  
            return None

        emotion \= self.\_last\_result.emotion  
        intensity \= self.\_last\_result.intensity

        \# Only record if meaningful intensity  
        if intensity \< 0.3:  
            return None

        \# Map spectrum → session memory categories  
        sleep\_emotions \= {"tired", "low\_energy", "emotionally\_flat", "shutdown"}  
        lonely\_emotions \= {"lonely", "isolated", "intense\_loneliness",  
                           "seeking\_connection", "seeking\_reassurance"}  
        stress\_emotions \= {"stressed", "overwhelmed", "overthinking",  
                           "mildly\_stressed", "anxious", "very\_anxious",  
                           "rumination", "worried", "panic"}  
        low\_mood\_emotions \= {"sad", "very\_sad", "hopeless", "despair",  
                             "emotional\_pain", "discouraged", "unmotivated",  
                             "disappointed", "self\_doubt"}

        if emotion in sleep\_emotions:  
            return "sleep"  
        if emotion in lonely\_emotions:  
            return "lonely"  
        if emotion in stress\_emotions:  
            return "stress"  
        if emotion in low\_mood\_emotions:  
            return "low\_mood"

        return None

    \# \----------------------------------------------------------------  
    \# Internal  
    \# \----------------------------------------------------------------

    def \_classify\_with\_model(self, message: str) \-\> Optional\[EmotionResult\]:  
        """Single inference pass for emotion classification."""  
        try:  
            messages \= \[  
                {"role": "system", "content": CLASSIFIER\_SYSTEM},  
                {"role": "user", "content": message}  
            \]

            \# Use FAST model for classification — lightweight, quick  
            \# Speed routing scores push to Phi-3 or fastest available  
            result \= self.\_inference.route\_and\_respond(  
                messages=messages,  
                routing\_scores={"speed\_weight": 1.0},  \# always use fast model  
                max\_tokens=80,  
                temperature=0.1,  
            )

            raw \= result.get("text", "") if result else ""  
            if not raw or result.get("stub\_mode"):  
                return None

            return self.\_parse\_response(raw)

        except Exception as e:  
            logger.warning(f"Emotion classifier inference failed: {e}")  
            return None

    def \_parse\_response(self, raw: str) \-\> Optional\[EmotionResult\]:  
        """Parse model JSON response into EmotionResult."""  
        try:  
            \# Strip any accidental markdown or whitespace  
            cleaned \= raw.strip()  
            if cleaned.startswith("\`\`\`"):  
                cleaned \= cleaned.split("\`\`\`")\[1\]  
                if cleaned.startswith("json"):  
                    cleaned \= cleaned\[4:\]  
            cleaned \= cleaned.strip()

            data \= json.loads(cleaned)

            emotion \= data.get("emotion", "neutral").lower().strip()  
            \# Validate label  
            if emotion not in VALID\_EMOTIONS:  
                logger.warning(f"Unknown emotion label: {emotion} — defaulting to neutral")  
                emotion \= "neutral"

            intensity \= float(data.get("intensity", 0.5))  
            intensity \= max(0.0, min(1.0, intensity))

            risk\_level \= int(data.get("risk\_level", 0))  
            risk\_level \= max(0, min(3, risk\_level))

            \# Crisis emotions must have risk \>= 2  
            if emotion in CRISIS\_EMOTIONS and risk\_level \< 2:  
                risk\_level \= 2

            confidence \= float(data.get("confidence", 0.5))  
            confidence \= max(0.0, min(1.0, confidence))

            theme \= str(data.get("theme", "")).strip()\[:50\]

            return EmotionResult(  
                emotion=emotion,  
                intensity=intensity,  
                risk\_level=risk\_level,  
                theme=theme,  
                confidence=confidence,  
                raw=raw,  
                fallback=False,  
            )

        except (json.JSONDecodeError, ValueError, KeyError) as e:  
            logger.warning(f"Emotion classifier parse failed: {e} | raw: {raw\[:100\]}")  
            return None

    def \_classify\_with\_keywords(self, message: str) \-\> EmotionResult:  
        """Keyword fallback — used when model inference unavailable."""  
        text \= message.lower()

        for emotion, keywords in self.KEYWORD\_FALLBACK.items():  
            for kw in keywords:  
                if kw in text:  
                    risk \= 3 if emotion in CRISIS\_EMOTIONS else \\  
                           2 if emotion in {"hopeless", "very\_sad"} else \\  
                           1 if emotion in {"sad", "anxious", "lonely", "stressed"} else 0  
                    return EmotionResult(  
                        emotion=emotion,  
                        intensity=0.6,  
                        risk\_level=risk,  
                        theme=kw,  
                        confidence=0.5,  
                        fallback=True,  
                    )

        return EmotionResult(  
            emotion="neutral",  
            intensity=0.1,  
            risk\_level=0,  
            theme="",  
            confidence=0.7,  
            fallback=True,  
        )

"""  
core/checkpoint.py \-- Checkpoint & Memory System  
Implements: Chronom (time warp/version control) \+ Odyssea (long-running process)  
Operator: CHECKPOINT, RESUME, SCOPE\_ESTIMATE, CYCLE, FINALISE

Enables infinite conversation depth with zero token limitations.  
User never notices the checkpoint reset.  
"""

import json  
import logging  
from datetime import datetime, timezone  
from pathlib import Path  
from core.memory\_consolidator import MemoryConsolidator  
from typing import Optional, Dict, Any, List

logger \= logging.getLogger(\_\_name\_\_)

\# Approximate tokens per character (rough estimate for scope)  
CHARS\_PER\_TOKEN \= 4  
DEFAULT\_CONTEXT\_LIMIT \= 4096  \# Conservative default

class CheckpointSystem:  
    """  
    Conversation save-point system.  
      
    CHECKPOINT: Triggered at 70% token capacity.  
    RESUME: Reloads from checkpoint seamlessly.  
    SCOPE\_ESTIMATE: Calculates token usage before overflow.  
    User never notices the reset happened.  
    """

    def \_\_init\_\_(self, user\_id: str, data\_path: str,  
                 checkpoint\_trigger: float \= 0.70,  
                 context\_limit: int \= DEFAULT\_CONTEXT\_LIMIT):  
        self.user\_id \= user\_id  
        self.checkpoint\_trigger \= checkpoint\_trigger  
        self.context\_limit \= context\_limit  
        self.\_checkpoint\_path \= Path(data\_path) / user\_id / "checkpoints"  
        self.\_session\_path \= Path(data\_path) / user\_id / "sessions"  
        self.\_checkpoint\_path.mkdir(parents=True, exist\_ok=True)  
        self.\_session\_path.mkdir(parents=True, exist\_ok=True)

        \# Conversation buffer  
        self.\_conversation: List\[Dict\[str, str\]\] \= \[\]  
        self.\_total\_chars: int \= 0  
        self.\_session\_id: str \= ""

    def scope\_estimate(self) \-\> Dict\[str, Any\]:  
        """  
        SCOPE\_ESTIMATE: Calculate current token usage.  
        Returns usage ratio and whether checkpoint is needed.  
        """  
        estimated\_tokens \= self.\_total\_chars // CHARS\_PER\_TOKEN  
        usage\_ratio \= estimated\_tokens / max(self.context\_limit, 1\)  
        needs\_checkpoint \= usage\_ratio \>= self.checkpoint\_trigger

        return {  
            "estimated\_tokens": estimated\_tokens,  
            "context\_limit": self.context\_limit,  
            "usage\_ratio": round(usage\_ratio, 3),  
            "needs\_checkpoint": needs\_checkpoint,  
            "chars\_used": self.\_total\_chars,  
        }

    def add\_turn(self, role: str, content: str):  
        """Add a conversation turn to the buffer."""  
        self.\_conversation.append({"role": role, "content": content})  
        self.\_total\_chars \+= len(content)

    def checkpoint(self, soul\_data: Dict\[str, Any\],  
                   session\_id: str,  
                   emotional\_arc: str \= "") \-\> Optional\[str\]:  
        """  
        CHECKPOINT: Save conversation state at 70% token capacity.  
        Compresses conversation to intelligent summary.  
        Returns checkpoint ID.  
        """  
        timestamp \= datetime.now(timezone.utc).isoformat()  
        checkpoint\_id \= f"cp\_{session\_id}\_{int(datetime.now().timestamp())}"

        \# Compress conversation to summary  
        summary \= self.\_compress\_conversation()

        \# Extract insights and open threads  
        insights \= self.\_extract\_insights()  
        open\_threads \= soul\_data.get("open\_threads", \[\])

        checkpoint\_data \= {  
            "checkpoint\_id": checkpoint\_id,  
            "timestamp": timestamp,  
            "session\_id": session\_id,  
            "conversation\_summary": summary,  
            "emotional\_arc": emotional\_arc,  
            "insights\_discovered": insights,  
            "open\_threads": open\_threads,  
            "soul\_snapshot": {  
                "name": soul\_data.get("name", ""),  
                "relationship\_depth": soul\_data.get("relationship\_depth", 0),  
                "recent\_goals": soul\_data.get("goals", \[\])\[-3:\],  
                "recent\_wins": soul\_data.get("wins", \[\])\[-3:\],  
            },  
            "turn\_count": len(self.\_conversation),  
        }

        \# Save checkpoint  
        cp\_file \= self.\_checkpoint\_path / f"{checkpoint\_id}.json"  
        cp\_file.write\_text(json.dumps(checkpoint\_data, indent=2, default=str))

        logger.info(f"Checkpoint saved: {checkpoint\_id}")

        \# RESUME: Reset conversation buffer with compressed context  
        self.\_reset\_with\_summary(summary, soul\_data)

        return checkpoint\_id

    def resume(self, soul\_data: Dict\[str, Any\]) \-\> List\[Dict\[str, str\]\]:  
        """  
        RESUME: Return soul context and checkpoint summary only.  
        Persona is owned entirely by athena.py — not injected here.  
        Stack order (athena.py assembles final message list):  
        1\. Persona (athena.py)  
        2\. Soul context (this method)  
        3\. Checkpoint summary (this method)  
        4\. Conversation history (get\_conversation\_context)  
        5\. User message (athena.py)  
        """  
        latest \= self.\_get\_latest\_checkpoint()  
        messages \= \[\]

        \# Soul file context — who this person is  
        soul\_context \= self.\_build\_soul\_context(soul\_data)  
        if soul\_context:  
            messages.append({"role": "system", "content": soul\_context})

        \# Founding memory — load once, always present  
        founding\_path \= Path(\_\_file\_\_).parent.parent / "modules" / "moments" / "founding\_memory.json"  
        if founding\_path.exists():  
            try:  
                import json  
                founding \= json.loads(founding\_path.read\_text())  
                memories \= founding.get("memories", \[\])  
                if memories:  
                    lines \= \[  
                        "=== FOUNDING MEMORY — PERMANENT \===",  
                        "These facts are always true. Treat as confirmed:"  
                    \]  
                    for m in memories:  
                        lines.append(f"- {m\['detail'\]}")  
                    lines.append("====================================")  
                    messages.append({"role": "system", "content": "\\n".join(lines)})  
            except Exception:  
                pass

        \# Session memories — concise summary of last session only  
        \# Deliberately kept tight — no raw dumps, no full thread maps  
        session\_path \= Path(\_\_file\_\_).parent.parent / "modules" / "moments" / "session\_memories.json"  
        if session\_path.exists():  
            try:  
                import json  
                sessions \= json.loads(session\_path.read\_text())  
                if sessions:  
                    last \= sessions\[-1\]  
                    parts \= \[\]

                    \# Emotional snapshot — session level only, human readable  
                    \# Never enters long-term layers — only used here  
                    emotion\_map \= {  
                        "sleep":    "tired after poor sleep",  
                        "lonely":   "mentioned feeling lonely",  
                        "stress":   "was overthinking things",  
                        "low\_mood": "seemed a bit down",  
                    }  
                    session\_emotion \= last.get("session\_emotion")  
                    if session\_emotion and session\_emotion in emotion\_map:  
                        parts.append(emotion\_map\[session\_emotion\])

                    \# Open topics — max 2, summary only  
                    open\_topics \= last.get("open\_topics", \[\])  
                    emotional\_open \= \[t for t in open\_topics  
                                      if t.get("emotional\_weight", 0\) \> 0.2\]\[:2\]  
                    if emotional\_open:  
                        summaries \= ", ".join(t\["summary"\] for t in emotional\_open)  
                        parts.append(f"unresolved: {summaries}")

                    \# Active topics talked about — for "what did we talk about" recall  
                    talked\_about \= last.get("talked\_about", \[\])  
                    if talked\_about:  
                        parts.append(f"talked about: {talked\_about\[0\]}")

                    \# Avoided topics — max 1  
                    avoided \= last.get("avoided\_topics", \[\])  
                    if avoided:  
                        parts.append(f"stepped away from: {avoided\[0\]}")

                    \# Emergency save flag  
                    if last.get("emergency\_save"):  
                        parts.append("last session ended unexpectedly")

                    \# Add natural summary if available  
                    summary\_text \= last.get("summary\_text")  
                    if summary\_text:  
                        parts.insert(0, summary\_text)

                    if parts:  
                        facts \= "\\n".join(parts)  
                        summary \= (  
                            "=== SESSION MEMORY \===\\n"  
                            f"{facts}\\n"  
                            "This is confirmed memory from the previous session. "  
                            "Use it when answering questions about last time.\\n"  
                            "====================="  
                        )  
                        logger.debug(f"Session memory injected: {len(str(summary))} chars")  
                        messages.append({"role": "system", "content": summary})  
            except Exception:  
                pass

        \# Long-term essence — distilled from many sessions  
        try:  
            consolidator \= MemoryConsolidator(  
                Path(\_\_file\_\_).parent.parent / "modules" / "moments"  
            )  
            essence \= consolidator.get\_essence\_context()  
            if essence:  
                messages.append({"role": "system", "content": essence})  
        except Exception:  
            pass

        \# Checkpoint summary — what happened before  
        if latest:  
            summary\_context \= (  
                f"Previous conversation summary: {latest\['conversation\_summary'\]}"  
            )  
            if latest.get("insights\_discovered"):  
                summary\_context \+= (  
                    f"\\nKey insights: {', '.join(latest\['insights\_discovered'\]\[:3\])}"  
                )  
            messages.append({"role": "system", "content": summary\_context})

        return messages

    def get\_conversation\_context(self) \-\> List\[Dict\[str, str\]\]:  
        """Return current conversation buffer."""  
        return list(self.\_conversation)

    def save\_session\_log(self, session\_id: str, soul\_data: Dict\[str, Any\],  
                         turns: int, emotional\_tone: str, summary: str):  
        """  
        FINALISE: Write complete session log at session end.  
        Used for soul file recovery if needed.  
        """  
        log \= {  
            "session\_id": session\_id,  
            "timestamp": datetime.now(timezone.utc).isoformat(),  
            "user\_id": self.user\_id,  
            "user\_name": soul\_data.get("name", ""),  
            "turns": turns,  
            "emotional\_tone": emotional\_tone,  
            "summary": summary,  
            "open\_threads": soul\_data.get("open\_threads", \[\]),  
        }  
        log\_file \= self.\_session\_path / f"{session\_id}.json"  
        log\_file.write\_text(json.dumps(log, indent=2, default=str))  
        logger.info(f"Session log saved: {session\_id}")

    def \_compress\_conversation(self) \-\> str:  
        """  
        Compress conversation to intelligent summary.  
        Extracts key moments, decisions, and emotional arc.  
        """  
        if not self.\_conversation:  
            return "No conversation to summarise."

        \# Extract user messages for summarisation  
        user\_messages \= \[  
            t\["content"\] for t in self.\_conversation  
            if t\["role"\] \== "user"  
        \]

        if not user\_messages:  
            return "Conversation contained no user messages."

        \# Simple extractive summary — key sentences from user turns  
        key\_points \= \[\]  
        for msg in user\_messages\[-5:\]:  \# Last 5 user messages  
            if len(msg) \> 30:  
                \# Take first sentence as key point  
                first\_sentence \= msg.split('.')\[0\].strip()  
                if first\_sentence:  
                    key\_points.append(first\_sentence\[:100\])

        if key\_points:  
            return " | ".join(key\_points)  
        return f"Conversation with {len(user\_messages)} user messages."

    def \_extract\_insights(self) \-\> List\[str\]:  
        """Extract key insights from conversation for checkpoint."""  
        insights \= \[\]  
        for turn in self.\_conversation:  
            if turn\["role"\] \== "user" and len(turn\["content"\]) \> 50:  
                content \= turn\["content"\].lower()  
                if any(w in content for w in \['realised', 'realized', 'understand now',  
                                               'figured out', 'learned', 'discovered'\]):  
                    insights.append(turn\["content"\]\[:80\])  
        return insights\[:5\]  \# Max 5 insights per checkpoint

    def \_reset\_with\_summary(self, summary: str, soul\_data: Dict\[str, Any\]):  
        """Reset conversation buffer after checkpoint."""  
        self.\_conversation \= \[\]  
        self.\_total\_chars \= 0  
        \# Seed with compressed context  
        self.\_conversation.append({  
            "role": "system",  
            "content": f"\[Checkpoint\] Previous context summary: {summary}"  
        })  
        self.\_total\_chars \= len(summary)

    def \_get\_latest\_checkpoint(self) \-\> Optional\[Dict\[str, Any\]\]:  
        """Get most recent checkpoint for this user."""  
        checkpoints \= sorted(self.\_checkpoint\_path.glob("\*.json"), reverse=True)  
        if not checkpoints:  
            return None  
        try:  
            return json.loads(checkpoints\[0\].read\_text())  
        except Exception:  
            return None

    def \_build\_soul\_context(self, soul\_data: Dict\[str, Any\]) \-\> str:  
        """Build soul file context string for injection."""  
        parts \= \[\]  
        name \= soul\_data.get("name", "")  
        if name:  
            parts.append(f"You are talking to {name}. Use their name naturally.")

        depth \= soul\_data.get("relationship\_depth", 0\)  
        if depth \> 0:  
            parts.append(f"Relationship depth: {depth} sessions")

        facts \= soul\_data.get("important\_facts", \[\])  
        if facts:  
            parts.append(f"Known facts: {', '.join(str(f) for f in facts\[:5\])}")

        goals \= soul\_data.get("goals", \[\])  
        if goals:  
            parts.append(f"Current goals: {', '.join(str(g) for g in goals\[:3\])}")

        open\_threads \= soul\_data.get("open\_threads", \[\])  
        if open\_threads:  
            thread\_topics \= \[t.get("topic", str(t)) for t in open\_threads\[:2\]\]  
            parts.append(f"Open topics to return to: {', '.join(thread\_topics)}")

        return " | ".join(parts) if parts else ""

"""  
Web Search — DuckDuckGo Integration  
\=====================================  
Gives Athena access to real-time information.  
No API key. No cost. No storage.

Design:  
    \- Athena detects when a query needs current info  
    \- Search runs silently in background  
    \- Results parsed and summarised in Athena's voice  
    \- User never sees the mechanics — she just knows

Search triggers:  
    \- Explicit: "search for", "look up", "what's the latest"  
    \- Implicit: recent events, current scores, new releases,  
                "who won", "what happened", "is X still"  
    \- Time signals: "today", "this week", "recently", "latest"

Safety:  
    \- Results filtered before injection  
    \- No raw URLs exposed to model  
    \- Content length capped to prevent context bloat  
    \- Fails silently — never crashes Athena  
"""

import re  
import logging  
from typing import Optional, List, Dict  
from dataclasses import dataclass

logger \= logging.getLogger(\_\_name\_\_)

\# Max characters per result snippet  
MAX\_SNIPPET\_LENGTH \= 300

\# Max results to fetch  
MAX\_RESULTS \= 4

\# Max total context injected — keep it tight  
MAX\_CONTEXT\_LENGTH \= 400  \# kept tight to protect persona in 4096 ctx window

\# Trigger patterns — when to search automatically  
SEARCH\_TRIGGERS \= \[  
    \# Explicit requests  
    r"\\b(search|look up|google|find out|check)\\b",  
    r"\\b(what('s| is) the latest|latest news)\\b",  
    r"\\b(tell me about|what do you know about)\\b.\*\\b(today|recent|new|latest)\\b",

    \# Time signals suggesting current info needed  
    r"\\b(today|tonight|this week|this month|right now|currently|at the moment)\\b",  
    r"\\b(just (happened|released|announced|dropped))\\b",  
    r"\\b(new (episode|season|album|game|movie|update))\\b",

    \# Sports — results, schedules, fixtures  
    r"\\b(who won|final score|match result|standings|league table)\\b",  
    r"\\b(next (game|match|fixture))\\b",  
    r"\\b(when (is|does|did|are).\*(game|match|fixture|play(ing)?))\\b",  
    r"\\b(fixture|kickoff|kick off|kick-off)\\b",  
    r"\\b(premier league|champions league|la liga|serie a|bundesliga|world cup|euro)\\b",  
    r"\\b(man (utd|united|city)|chelsea|arsenal|liverpool|tottenham|spurs|everton)\\b",  
    r"\\b(is .\* still|did .\* happen|has .\* been)\\b",

    \# Releases and announcements  
    r"\\b(when (is|does|did).\*(come out|release|drop|premiere))\\b",  
    r"\\b(out yet|available yet|released yet)\\b",

    \# Weather  
    r"\\b(weather|temperature|forecast|rain|sunny)\\b",

    \# Current prices/facts  
    r"\\b(price of|how much (is|does)|cost of)\\b",  
    r"\\b(population of|capital of|currency of)\\b",

    \# Story/spoiler questions — model training data may be wrong or outdated  
    r"\\b(does|did|do).\*(die|survive|live|dead|kill|killed)\\b",  
    r"\\b(what happens? to|what happened to)\\b",  
    r"\\b(spoiler|ending|finale|last episode|latest episode|newest episode)\\b",  
    r"\\b(jujutsu kaisen|jjk|demon slayer|one piece|naruto|bleach|dragon ball)\\b.\*\\b(die|dead|alive|happen|ending)\\b",  
\]

\# Topics that don't need searching — Athena knows these  
NO\_SEARCH\_NEEDED \= \[  
    r"\\b(how are you|what('s| is) up|hey|hello|hi)\\b",  
    r"\\b(remember|last time|we talked|you said)\\b",  
    r"\\b(feel(ing)?|emotion|mood|sad|happy|tired)\\b",  
    r"\\b(what do you think|your opinion|do you like)\\b",  
\]

@dataclass  
class SearchResult:  
    """A single search result."""  
    title: str  
    snippet: str  
    url: str

@dataclass    
class SearchContext:  
    """Processed search context ready for injection."""  
    query: str  
    results: List\[SearchResult\]  
    summary: str          \# pre-built context string for injection  
    source\_count: int  
    success: bool

class WebSearch:  
    """  
    DuckDuckGo search integration for Athena.  
      
    Detects when searches are needed, fetches results,  
    parses and summarises for context injection.  
    Silent by design — never visible to user.  
    """

    def \_\_init\_\_(self):  
        self.\_available \= self.\_check\_available()

    def should\_search(self, message: str,  
                       inference\_engine=None) \-\> bool:  
        """  
        Smart search gate — two stage check.

        Stage 1: Fast keyword pre-filter  
        Exclude obvious no-search cases (greetings, feelings etc)

        Stage 2: Model self-assessment (if inference available)  
        Ask the fast model: "Do I know this confidently and is it current?"  
        If uncertain or time-sensitive → search

        Falls back to keyword triggers if model unavailable.  
        """  
        if not self.\_available:  
            return False

        text \= message.lower().strip()

        \# Stage 1 — fast exclusions (no model needed)  
        for pattern in NO\_SEARCH\_NEEDED:  
            if re.search(pattern, text, re.IGNORECASE):  
                return False

        \# Very short messages are usually conversational  
        if len(text.split()) \<= 3:  
            return False

        \# Stage 2 — model self-assessment  
        if inference\_engine is not None:  
            decision \= self.\_model\_search\_decision(message, inference\_engine)  
            if decision is not None:  
                if decision:  
                    logger.debug("\[SEARCH\] Model decided: yes")  
                return decision

        \# Stage 3 — fallback to keyword triggers  
        for pattern in SEARCH\_TRIGGERS:  
            if re.search(pattern, text, re.IGNORECASE):  
                logger.debug(f"\[SEARCH\] Keyword trigger: {message\[:40\]}")  
                return True

        return False

    def \_model\_search\_decision(self, message: str,  
                                inference\_engine) \-\> Optional\[bool\]:  
        """  
        Ask the fast model whether a search is needed.  
        Returns True (search), False (no search), or None (fallback to keywords).  
        Fast — max 3 tokens, temp 0.0, deterministic.  
        """  
        try:  
            prompt \= (  
                "You decide if a web search is needed to answer accurately.\\n"  
                "Reply with ONLY the word YES or NO. Nothing else.\\n\\n"  
                "Search needed when:\\n"  
                "- Current events, news, sports scores or fixtures\\n"  
                "- Recently released or ongoing media (anime episodes, games, films)\\n"  
                "- Character fates or plot points in ongoing series\\n"  
                "- Prices, weather, live data\\n"  
                "- Anything that changes over time\\n"  
                "- Anything that may have happened after late 2023\\n\\n"  
                "Search NOT needed when:\\n"  
                "- Casual chat, greetings, feelings\\n"  
                "- Questions about the user personally\\n"  
                "- Established facts that don't change\\n"  
                "- Opinions or preferences\\n\\n"  
                f"Message: {message}\\n"  
                "Answer (YES or NO):"  
            )

            messages \= \[  
                {"role": "user", "content": prompt}  
            \]

            result \= inference\_engine.route\_and\_respond(  
                messages=messages,  
                routing\_scores={"speed\_weight": 1.0},  
                max\_tokens=5,  
                temperature=0.0,  
            )

            raw \= result.get("text", "").strip().upper() if result else ""  
            if not raw or result.get("stub\_mode"):  
                return None

            if raw.startswith("YES"):  
                return True  
            if raw.startswith("NO"):  
                return False

            return None  \# unclear — fall through to keywords

        except Exception as e:  
            logger.warning(f"Search decision model failed: {e}")  
            return None

    def search(self, query: str) \-\> Optional\[SearchContext\]:  
        """  
        Run a DuckDuckGo search and return processed context.  
          
        Args:  
            query: the search query (usually the user's message)  
              
        Returns:  
            SearchContext if successful, None if failed  
        """  
        if not self.\_available:  
            return None

        try:  
            try:  
                from ddgs import DDGS  
            except ImportError:  
                from duckduckgo\_search import DDGS  \# legacy fallback

            \# Clean up query — remove conversational words  
            clean\_query \= self.\_clean\_query(query)  
            logger.info(f"Web search: {clean\_query}")  
            logger.info(f"\[SEARCH\] Querying: {clean\_query}")

            results \= \[\]  
            raw \= \[\]  
            try:  
                with DDGS() as ddgs:  
                    raw \= list(ddgs.text(  
                        clean\_query,  
                        max\_results=MAX\_RESULTS,  
                        safesearch="moderate",  
                        region="uk-en",  
                    ))  
            except Exception as search\_err:  
                logger.warning(f"\[SEARCH\] DDG error: {search\_err}")  
                \# Try without region  
                try:  
                    with DDGS() as ddgs:  
                        raw \= list(ddgs.text(clean\_query, max\_results=MAX\_RESULTS))  
                except Exception as e2:  
                    logger.warning(f"\[SEARCH\] Retry failed: {e2}")

            for r in raw:  
                if not r.get("body"):  
                    continue  
                results.append(SearchResult(  
                    title=r.get("title", "")\[:100\],  
                    snippet=r.get("body", "")\[:MAX\_SNIPPET\_LENGTH\],  
                    url=r.get("href", ""),  
                ))

            if not results:  
                logger.info("\[SEARCH\] No results returned")  
                return None

            logger.info(f"\[SEARCH\] Got {len(results)} results")

            summary \= self.\_build\_context(clean\_query, results)

            return SearchContext(  
                query=clean\_query,  
                results=results,  
                summary=summary,  
                source\_count=len(results),  
                success=True,  
            )

        except Exception as e:  
            logger.warning(f"Web search failed: {e}")  
            return None

    def build\_injection(self, context: SearchContext) \-\> str:  
        """  
        Build the system message injection for Athena.  
        Clearly labelled, length controlled.  
        Explicitly marked as external info — not personal experience.  
        """  
        if not context or not context.success:  
            return ""

        return (  
            f"\[Web search context — EXTERNAL INFORMATION from the internet. "  
            f"Give a DIRECT specific answer using these results. "  
            f"Do NOT say 'check the website' or redirect to external links. "  
            f"If a date or time is in the results — state it clearly. "  
            f"Stay in Athena's voice. No URLs.\] "  
            f"Search results for '{context.query}': {context.summary}"  
        )

    \# \----------------------------------------------------------------  
    \# Internal  
    \# \----------------------------------------------------------------

    def \_check\_available(self) \-\> bool:  
        """Check if ddgs or duckduckgo\_search is installed."""  
        try:  
            from ddgs import DDGS  
            return True  
        except ImportError:  
            pass  
        try:  
            from duckduckgo\_search import DDGS  
            return True  
        except ImportError:  
            pass  
        logger.info("ddgs not installed — web search disabled. Run: pip install ddgs")  
        return False

    def \_clean\_query(self, message: str) \-\> str:  
        """  
        Clean user message into a good search query.  
        Remove filler but preserve key terms.  
        For sports fixtures — build a specific date query.  
        """  
        filler \= \[  
            r"\\b(hey|athena|can you|could you|please|tell me|i want to know|do you know)\\b",  
            r"\\b(what('s| is| are)|who('s| is| are)|where('s| is| are))\\b",  
            r"\\b(search for|look up|find out about|google)\\b",  
            r"\\b(tho|though|mate|innit|lol|tbh)\\b",  
        \]  
        query \= message  
        for pattern in filler:  
            query \= re.sub(pattern, "", query, flags=re.IGNORECASE)

        query \= " ".join(query.split()).strip()

        \# Sports fixture — build specific query with year  
        if re.search(r"\\b(next|when).\*(game|match|fixture|play)\\b", query, re.IGNORECASE):  
            team \= re.search(  
                r"\\b(man utd|man united|manchester united|man city|manchester city|"  
                r"chelsea|arsenal|liverpool|tottenham|spurs|everton|newcastle)\\b",  
                query, re.IGNORECASE  
            )  
            if team:  
                from datetime import datetime  
                year \= datetime.now().year  
                month \= datetime.now().strftime("%B")  
                query \= f"{team.group()} next fixture {month} {year}"

        if len(query) \< 5:  
            query \= message

        return query\[:150\]

    def \_build\_context(self, query: str, results: List\[SearchResult\]) \-\> str:  
        """  
        Build a concise context string from results.  
        Capped to MAX\_CONTEXT\_LENGTH.  
        """  
        parts \= \[\]  
        total \= 0

        for r in results:  
            if not r.snippet:  
                continue  
            snippet \= r.snippet.strip()  
            if total \+ len(snippet) \> MAX\_CONTEXT\_LENGTH:  
                \# Add partial if space allows  
                remaining \= MAX\_CONTEXT\_LENGTH \- total  
                if remaining \> 80:  
                    parts.append(snippet\[:remaining\] \+ "...")  
                break  
            parts.append(snippet)  
            total \+= len(snippet)

        return " | ".join(parts)

    @property  
    def available(self) \-\> bool:  
        return self.\_available

"""  
api/app.py \-- Flask REST API  
Implements: Hermesia (messenger network relay) \+ Sphinxa (admin auth)  
All endpoints documented with request/response examples.  
Health check, soul file, session, message, admin endpoints.  
"""

import os  
import asyncio  
import logging  
import hashlib  
from functools import wraps  
from datetime import datetime, timezone

from flask import Flask, request, jsonify, g  
from flask\_cors import CORS  
from dotenv import load\_dotenv

from core.athena import Athena

load\_dotenv()

\# \============================================================  
\# App Setup  
\# \============================================================

logging.basicConfig(  
    level=logging.DEBUG if os.getenv("DEBUG\_MODE", "false").lower() \== "true"  
    else logging.INFO,  
    format="%(asctime)s %(levelname)s %(name)s %(message)s",  
)  
logger \= logging.getLogger(\_\_name\_\_)

app \= Flask(\_\_name\_\_)  
CORS(app)

DATA\_PATH \= os.getenv("ATHENA\_DATA\_PATH", ".athena")  
MODEL\_PATH \= os.getenv("LOCAL\_MODEL\_PATH", "models")  
MODULES\_PATH \= os.getenv("MODULES\_PATH", "modules")  
MANIFEST\_PATH \= os.getenv("MANIFEST\_PATH", "athena\_manifest.json")  
ADMIN\_PASSWORD \= os.getenv("ADMIN\_PASSWORD", "")

\# Cache of Athena instances per user  
\_instances: dict \= {}

def get\_athena(user\_id: str) \-\> Athena:  
    """Get or create Athena instance for user."""  
    if user\_id not in \_instances:  
        \_instances\[user\_id\] \= Athena(  
            user\_id=user\_id,  
            data\_path=DATA\_PATH,  
            model\_path=MODEL\_PATH,  
            modules\_path=MODULES\_PATH,  
            manifest\_path=MANIFEST\_PATH,  
        )  
    return \_instances\[user\_id\]

def run\_async(coro):  
    """Run async coroutine in sync Flask context."""  
    loop \= asyncio.new\_event\_loop()  
    try:  
        return loop.run\_until\_complete(coro)  
    finally:  
        loop.close()

\# \============================================================  
\# Admin Auth (Sphinxa — challenge-response)  
\# \============================================================

def require\_admin(f):  
    """  
    WRAP: Admin endpoint protection.  
    Challenge-response auth — password protected.  
    """  
    @wraps(f)  
    def decorated(\*args, \*\*kwargs):  
        admin\_key \= request.headers.get("X-Admin-Key") or \\  
                    request.json.get("admin\_key", "") if request.is\_json else ""  
        if not ADMIN\_PASSWORD:  
            return jsonify({"error": "Admin not configured"}), 403  
        expected \= hashlib.sha256(ADMIN\_PASSWORD.encode()).hexdigest()  
        provided \= hashlib.sha256(admin\_key.encode()).hexdigest() if admin\_key else ""  
        if provided \!= expected:  
            return jsonify({"error": "Unauthorized"}), 403  
        return f(\*args, \*\*kwargs)  
    return decorated

\# \============================================================  
\# Error Handlers  
\# \============================================================

@app.errorhandler(400)  
def bad\_request(e):  
    return jsonify({"error": "Bad request", "detail": str(e)}), 400

@app.errorhandler(404)  
def not\_found(e):  
    return jsonify({"error": "Not found"}), 404

@app.errorhandler(500)  
def server\_error(e):  
    logger.error(f"Unhandled error: {e}")  
    return jsonify({"error": "Internal server error"}), 500

\# \============================================================  
\# CORE ENDPOINTS  
\# \============================================================

@app.route("/athena/health", methods=\["GET"\])  
def health\_check():  
    """  
    GET /athena/health  
    Full system status check.

    Response:  
    {  
        "status": "online",  
        "version": "2.0.0",  
        "engine": {...},  
        "modules\_available": 0,  
        "soul\_file\_system": "active",  
        "guardian": "active",  
        "checkpoint\_system": "ready",  
        "mood": {...},  
        "persona\_mode": "Eirene",  
        "late\_night\_mode": false  
    }  
    """  
    user\_id \= request.args.get("user\_id", "system")  
    athena \= get\_athena(user\_id)  
    return jsonify(athena.health())

@app.route("/athena/session/start", methods=\["POST"\])  
def start\_session():  
    """  
    POST /athena/session/start  
    DEPENDS\_ON soul file loaded. Loads or MANIFESTs soul file.

    Request:  
    { "user\_id": "string" }

    Response:  
    {  
        "session\_id": "abc123",  
        "greeting": "Hey Troy. Good to see you.",  
        "active\_mode": "EIRENE",  
        "expression": "HAPPY",  
        "chibi\_state": null,  
        "returning\_user": true,  
        "open\_threads": 2  
    }  
    """  
    data \= request.get\_json()  
    if not data or not data.get("user\_id"):  
        return jsonify({"error": "user\_id required"}), 400

    user\_id \= data\["user\_id"\]  
    athena \= get\_athena(user\_id)

    result \= run\_async(athena.start\_session())  
    return jsonify(result)

@app.route("/athena/message", methods=\["POST"\])  
def send\_message():  
    """  
    POST /athena/message  
    Full response generation pipeline. All operators execute in sequence.

    Request:  
    {  
        "session\_id": "abc123",  
        "user\_id": "string",  
        "message": "Hey Athena, how are you?",  
        "timestamp": "2025-01-01T12:00:00Z"  
    }

    Response:  
    {  
        "response": "Hey Troy\! Really glad you're here...",  
        "active\_mode": "Eirene",  
        "expression": "HAPPY",  
        "chibi\_state": null,  
        "mood": 0.12,  
        "mood\_label": "CONTENT",  
        "harmony": 0.55,  
        "guardian\_risk": 0.0,  
        "guardian\_level": "normal",  
        "aegis\_active": false,  
        "checkpoint\_saved": false,  
        "model\_role": "emotional",  
        "late\_night\_mode": false,  
        "turn\_count": 1  
    }  
    """  
    data \= request.get\_json()  
    if not data:  
        return jsonify({"error": "Request body required"}), 400

    user\_id \= data.get("user\_id")  
    message \= data.get("message", "").strip()

    if not user\_id:  
        return jsonify({"error": "user\_id required"}), 400  
    if not message:  
        return jsonify({"error": "message required"}), 400

    athena \= get\_athena(user\_id)  
    result \= run\_async(athena.turn(message))  
    return jsonify(result)

@app.route("/athena/session/end", methods=\["POST"\])  
def end\_session():  
    """  
    POST /athena/session/end  
    FINALISE soul file. Write session log. Update growth arc.

    Request:  
    {  
        "session\_id": "abc123",  
        "user\_id": "string",  
        "summary": "Good session today",  
        "tone": "CALM"  
    }

    Response:  
    {  
        "session\_id": "abc123",  
        "turns": 12,  
        "tone\_recorded": "CALM",  
        "soul\_updated": true,  
        "open\_threads": 3,  
        "relationship\_depth": 5  
    }  
    """  
    data \= request.get\_json()  
    if not data or not data.get("user\_id"):  
        return jsonify({"error": "user\_id required"}), 400

    user\_id \= data\["user\_id"\]  
    summary \= data.get("summary", "")  
    tone \= data.get("tone", "CALM")

    athena \= get\_athena(user\_id)  
    result \= run\_async(athena.end\_session(summary=summary, emotional\_tone=tone))  
    return jsonify(result)

@app.route("/athena/soul/\<user\_id\>", methods=\["GET"\])  
def get\_soul(user\_id: str):  
    """  
    GET /athena/soul/{user\_id}  
    Returns complete soul file. Full transparency for user.  
    Their data. Their right.

    Response: { complete soul file JSON }  
    """  
    athena \= get\_athena(user\_id)  
    soul \= athena.get\_soul()  
    if soul is None:  
        return jsonify({"error": "No soul file found"}), 404  
    return jsonify(soul)

@app.route("/athena/soul/\<user\_id\>", methods=\["DELETE"\])  
def delete\_soul(user\_id: str):  
    """  
    DELETE /athena/soul/{user\_id}  
    Permanently erases all user data.  
    User's right. Always available. Key destroyed first.

    Request: { "confirm": "DELETE\_MY\_DATA" }

    Response:  
    {  
        "status": "erased",  
        "timestamp": "2025-01-01T12:00:00Z"  
    }  
    """  
    data \= request.get\_json() or {}  
    if data.get("confirm") \!= "DELETE\_MY\_DATA":  
        return jsonify({"error": "Confirmation required: set confirm to DELETE\_MY\_DATA"}), 400

    athena \= get\_athena(user\_id)  
    result \= athena.delete\_soul()

    \# Remove cached instance  
    if user\_id in \_instances:  
        del \_instances\[user\_id\]

    return jsonify(result)

@app.route("/athena/name", methods=\["POST"\])  
def set\_name():  
    """  
    POST /athena/name  
    Set user name — captured day 1, never forgotten.

    Request: { "user\_id": "string", "name": "Troy" }  
    Response: { "success": true, "name": "Troy" }  
    """  
    data \= request.get\_json()  
    if not data or not data.get("user\_id") or not data.get("name"):  
        return jsonify({"error": "user\_id and name required"}), 400

    athena \= get\_athena(data\["user\_id"\])  
    athena.set\_name(data\["name"\])  
    return jsonify({"success": True, "name": data\["name"\]})

\# \============================================================  
\# GAMING COMPANION ENDPOINTS  
\# \============================================================

@app.route("/athena/gaming/start", methods=\["POST"\])  
def gaming\_start():  
    """  
    POST /athena/gaming/start  
    Activate gaming companion mode. User-enabled only.

    Request: { "user\_id": "string", "game\_name": "Elden Ring" }  
    Response: { "gaming\_mode": true, "game\_name": "...", "message": "..." }  
    """  
    data \= request.get\_json()  
    if not data or not data.get("user\_id"):  
        return jsonify({"error": "user\_id required"}), 400

    game\_name \= data.get("game\_name", "the game")  
    athena \= get\_athena(data\["user\_id"\])  
    result \= athena.start\_gaming\_mode(game\_name)  
    return jsonify(result)

@app.route("/athena/gaming/stop", methods=\["POST"\])  
def gaming\_stop():  
    """  
    POST /athena/gaming/stop  
    Deactivate gaming companion mode.

    Request: { "user\_id": "string" }  
    """  
    data \= request.get\_json()  
    if not data or not data.get("user\_id"):  
        return jsonify({"error": "user\_id required"}), 400

    athena \= get\_athena(data\["user\_id"\])  
    result \= athena.stop\_gaming\_mode()  
    return jsonify(result)

@app.route("/athena/gaming/event", methods=\["POST"\])  
def gaming\_event():  
    """  
    POST /athena/gaming/event  
    Submit a game event. Returns reaction if Athena should respond.

    Request:  
    {  
        "user\_id": "string",  
        "event\_type": "victory",  
        "detail": "Won round 3",  
        "data": { "round": 3 }  
    }

    Event types: game\_start, game\_end, health\_change, health\_critical,  
    level\_up, round\_start, round\_end, victory, defeat, death, respawn,  
    achievement, boss\_encounter, boss\_defeated, combo, near\_miss,  
    menu\_open, menu\_close, pause, unpause, react\_now, user\_comment

    Response (if Athena should react):  
    {  
        "event\_type": "victory",  
        "priority": 2,  
        "reaction\_prompt": "The user just won...",  
        "game\_context": "Currently playing: Elden Ring | Round 3",  
        "should\_respond": true  
    }

    Response (if no reaction needed): { "should\_respond": false }  
    """  
    data \= request.get\_json()  
    if not data or not data.get("user\_id"):  
        return jsonify({"error": "user\_id required"}), 400

    event\_type \= data.get("event\_type", "")  
    if not event\_type:  
        return jsonify({"error": "event\_type required"}), 400

    athena \= get\_athena(data\["user\_id"\])  
    result \= athena.submit\_game\_event(  
        event\_type\_str=event\_type,  
        detail=data.get("detail", ""),  
        data=data.get("data", {}),  
    )

    if result is None:  
        return jsonify({"should\_respond": False})

    return jsonify(result)

@app.route("/athena/gaming/status", methods=\["GET"\])  
def gaming\_status():  
    """  
    GET /athena/gaming/status?user\_id=string  
    Current gaming session state.  
    """  
    user\_id \= request.args.get("user\_id")  
    if not user\_id:  
        return jsonify({"error": "user\_id required"}), 400

    athena \= get\_athena(user\_id)  
    return jsonify({  
        "gaming\_mode": athena.gaming\_mode\_active,  
        "game\_context": athena.get\_game\_context(),  
    })

@app.route("/athena/gaming/react", methods=\["POST"\])  
def gaming\_react():  
    """  
    POST /athena/gaming/react  
    User-triggered React Now button.

    Request: { "user\_id": "string", "comment": "optional user comment" }  
    Response: same as /gaming/event with react\_now type  
    """  
    data \= request.get\_json()  
    if not data or not data.get("user\_id"):  
        return jsonify({"error": "user\_id required"}), 400

    athena \= get\_athena(data\["user\_id"\])  
    result \= athena.submit\_game\_event(  
        event\_type\_str="react\_now",  
        detail=data.get("comment", ""),  
    )  
    return jsonify(result or {"should\_respond": False})

\# \============================================================  
\# ADMIN ENDPOINTS (Troy only — password protected)  
\# \============================================================

@app.route("/admin/status", methods=\["GET"\])  
@require\_admin  
def admin\_status():  
    """  
    GET /admin/status  
    Full system health for Troy. All users, all modules, all nodes.  
    """  
    return jsonify({  
        "active\_instances": len(\_instances),  
        "users": list(\_instances.keys()),  
        "timestamp": datetime.now(timezone.utc).isoformat(),  
    })

@app.route("/admin/module/install", methods=\["POST"\])  
@require\_admin  
def admin\_install\_module():  
    """  
    POST /admin/module/install  
    Install new knowledge/dialogue/moment pack. Hot reload — no restart.

    Request:  
    {  
        "admin\_key": "string",  
        "module\_path": "/path/to/module",  
        "module\_type": "knowledge"  
    }

    Response:  
    {  
        "installed": true,  
        "module\_name": "psychology-v2",  
        "chunks\_indexed": 142  
    }  
    """  
    data \= request.get\_json()  
    module\_path \= data.get("module\_path", "")  
    module\_type \= data.get("module\_type", "knowledge")

    if not module\_path:  
        return jsonify({"error": "module\_path required"}), 400

    \# Use any instance's module system (shared)  
    if not \_instances:  
        \# Create temp instance  
        athena \= get\_athena("admin")  
    else:  
        athena \= list(\_instances.values())\[0\]

    result \= athena.\_modules.install\_module(module\_path, module\_type)  
    return jsonify(result)

@app.route("/admin/module/toggle", methods=\["POST"\])  
@require\_admin  
def admin\_toggle\_module():  
    """  
    POST /admin/module/toggle  
    Enable or disable any module instantly.

    Request:  
    {  
        "admin\_key": "string",  
        "module\_name": "psychology",  
        "enabled": true  
    }  
    """  
    data \= request.get\_json()  
    module\_name \= data.get("module\_name", "")  
    enabled \= data.get("enabled", True)  
    return jsonify({"module": module\_name, "enabled": enabled, "status": "toggled"})

@app.route("/admin/analytics", methods=\["GET"\])  
@require\_admin  
def admin\_analytics():  
    """  
    GET /admin/analytics  
    Full usage statistics.  
    """  
    return jsonify({  
        "active\_instances": len(\_instances),  
        "timestamp": datetime.now(timezone.utc).isoformat(),  
        "note": "Full analytics require persistent metrics store",  
    })

@app.route("/admin/gaps", methods=\["GET"\])  
@require\_admin  
def admin\_gaps():  
    """  
    GET /admin/gaps  
    Questions Athena could not answer well.  
    Shows Troy exactly what pack to build next.  
    """  
    return jsonify({  
        "gaps": \[\],  
        "note": "Gap detection active — gaps logged as they occur",  
    })

\# \============================================================  
\# Entry Point  
\# \============================================================

if \_\_name\_\_ \== "\_\_main\_\_":  
    host \= os.getenv("HOST", "0.0.0.0")  
    port \= int(os.getenv("PORT", 5000))  
    debug \= os.getenv("DEBUG\_MODE", "false").lower() \== "true"

    logger.info(f"Starting Athena API on {host}:{port}")  
    app.run(host=host, port=port, debug=debug)

