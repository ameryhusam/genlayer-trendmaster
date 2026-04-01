# { "Depends": "py-genlayer:1jb45aa8ynh2a9c9xn3b7qqh8sm5q93hwfp7jqmwsfhh8jpz09h6" }

from genlayer import *

class TrendMaster(gl.Contract):
    current_trend: str
    trend_score: u8       # 0-100 (Strength of the trend)
    authenticity: str     # "Organic", "Manipulated", or "Fading"
    analysis: str

    def __init__(self):
        self.current_trend = "None"
        self.trend_score = u8(0)
        self.authenticity = "Unknown"
        self.analysis = "Awaiting execution..."

    @gl.public.write
    def process_trends(self, specific_topic: str = ""):
        # Step 1: Determine the Target
        if not specific_topic:
            search_query = "top trending global news headlines today"
        else:
            search_query = f"is {specific_topic} trending now news analysis"

        def leader_fn():
            # Step 2: Fetch broad data
            search_url = f"https://www.google.com/search?q={search_query.replace(' ', '+')}"
            raw_data = str(gl.nondet.web.get(search_url))
            
            prompt = """
            Based on these search results, perform a trend analysis:
            1. Identify the primary trending topic.
            2. Determine the 'Trend Score' (0-100) based on volume/frequency.
            3. Evaluate authenticity: Is this a 'Fake/Botted' trend, or a 'Continuous/Organic' trend?
            
            DATA: {data}
            
            Return JSON: 
            {"topic": "string", "score": 0-100, "status": "Organic|Fake|Fading", "summary": "string"}
            """
            
            return gl.nondet.exec_prompt(prompt.format(data=raw_data[:8000]), response_format="json")

        def validator_fn(leader_result) -> bool:
            if not isinstance(leader_result, gl.vm.Return): return False
            proposed = leader_result.calldata
            
            # Local node verification
            local_raw = str(gl.nondet.web.get(f"https://www.google.com/search?q={search_query.replace(' ', '+')}"))
            
            # Consensus check: Do the nodes agree on the 'status' (Organic vs Fake)?
            # This is the most critical part of the truth-machine.
            local_check = gl.nondet.exec_prompt("Is this trend organic or fake? " + local_raw[:4000])
            return proposed["status"].lower() in local_check.lower()

        try:
            result = gl.vm.run_nondet(leader_fn, validator_fn)
            self.current_trend = result["topic"]
            self.trend_score = u8(result["score"])
            self.authenticity = result["status"]
            self.analysis = result["summary"]
        except Exception as e:
            self.analysis = f"Consensus failed: Nodes could not agree on the trend validity. Error: {str(e)}"

    @gl.public.view
    def get_trend_report(self) -> dict:
        return {
            "Trend": self.current_trend,
            "Strength": f"{self.trend_score}/100",
            "Type": self.authenticity,
            "AI_Analysis": self.analysis
        }

