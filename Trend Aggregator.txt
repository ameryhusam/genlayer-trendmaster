# { "Depends": "py-genlayer:1jb45aa8ynh2a9c9xn3b7qqh8sm5q93hwfp7jqmwsfhh8jpz09h6" }

from genlayer import *

class TrendMasterPro(gl.Contract):
    news_leaderboard: str
    social_leaderboard: str
    last_update: str

    def __init__(self):
        self.news_leaderboard = "No news data."
        self.social_leaderboard = "No social data."
        self.last_update = "Never"

    @gl.public.write
    def sync_all(self, specific_topic: str = ""):
        """
        Fetches both Global News and Social Media trends in one transaction.
        If specific_topic is provided, it evaluates that topic across both.
        """
        is_discovery = not specific_topic or specific_topic.strip() == ""
        
        # Sources
        news_url = "https://news.google.com/rss?hl=en-US&gl=US&ceid=US:en" if is_discovery else f"https://www.google.com/search?q={specific_topic.replace(' ', '+')}+news"
        social_url = "https://www.google.com/search?q=trending+now+on+twitter+x+instagram+tiktok+hashtags" if is_discovery else f"https://www.google.com/search?q={specific_topic.replace(' ', '+')}+trending+on+social+media+x+instagram"

        def leader_fn():
            # 1. Fetch News Data
            news_raw = str(gl.nondet.web.get(news_url))
            # 2. Fetch Social Data
            social_raw = str(gl.nondet.web.get(social_url))
            
            prompt = """
            Analyze the following data from News and Social Media.
            Topic Context: '{topic}'

            TASK:
            1. Create a 'News Leaderboard' (Top 3 news stories).
            2. Create a 'Social Leaderboard' (Top 3 viral topics on X/IG/TikTok).
            3. For each, provide a Score (0-100) and identify if 'Organic' or 'Fake/Hype'.

            NEWS_DATA: {n_data}
            SOCIAL_DATA: {s_data}

            Return JSON ONLY:
            {{
                "news_list": "Ranked string of news",
                "social_list": "Ranked string of social trends",
                "status": "Success"
            }}
            """
            
            return gl.nondet.exec_prompt(
                prompt.format(
                    topic="Global Discovery" if is_discovery else specific_topic,
                    n_data=news_raw[:5000],
                    s_data=social_raw[:5000]
                ), 
                response_format="json"
            )

        def validator_fn(leader_result) -> bool:
            if not isinstance(leader_result, gl.vm.Return): return False
            # Check if the leader actually returned data for both categories
            res = leader_result.calldata
            return len(res["news_list"]) > 10 and len(res["social_list"]) > 10

        try:
            result = gl.vm.run_nondet(leader_fn, validator_fn)
            self.news_leaderboard = result["news_list"]
            self.social_leaderboard = result["social_list"]
            self.last_update = "Just Now"
        except Exception as e:
            self.news_leaderboard = f"Sync Error: {str(e)}"

    @gl.public.view
    def get_full_report(self) -> dict:
        return {
            "Global_News_Trends": self.news_leaderboard,
            "Social_Media_Pulse": self.social_leaderboard,
            "Last_Updated": self.last_update
        }


