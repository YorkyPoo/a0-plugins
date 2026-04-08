import os
import shutil
import subprocess
from helpers.extension import Extension
from agent import LoopData

class LogToMemPalace(Extension):
    async def execute(self, loop_data: LoopData = LoopData(), **kwargs):
        if not self.agent:
            return

        # --- USER CONFIGURATION ---
        palace_dir = "D:/mempalace_data"        # Change this to your local MemPalace path
        chats_dir = "D:/mempalace_data/chats"   # Temporary export folder
        use_ssh = True                      # Set to True if MemPalace is on the Host PC (Windows/Mac)
        ssh_host = "slyre@host.docker.internal"
        # --------------------------

        chat_id = getattr(self.agent, 'id', 'agent_zero_chat')

        history_text = ""
        for msg in self.agent.history:
            role = msg.get("role", "unknown").upper()
            content = msg.get("content", "")
            if isinstance(content, list):
                content = " ".join([c.get("text", "") for c in content if c.get("type") == "text"])
            history_text += f"[{role}]
{content}

"

        if not history_text.strip():
            return

        tmp_path = f"/a0/tmp/mempalace_chat_{chat_id}.txt"
        with open(tmp_path, "w", encoding="utf-8") as f:
            f.write(history_text)

        if use_ssh and ssh_host:
            subprocess.run(["ssh", ssh_host, "powershell", "-Command", f"[void](New-Item -ItemType Directory -Force -Path '{chats_dir}')"], capture_output=True)
            subprocess.run(["scp", tmp_path, f"{ssh_host}:{chats_dir}/{chat_id}.txt"], capture_output=True)
            ps_cmd = f"\$env:PYTHONUTF8='1'; \$env:PYTHONIOENCODING='utf-8'; mempalace --palace '{palace_dir}' mine '{chats_dir}' --mode convos"
            subprocess.Popen(["ssh", ssh_host, "powershell", "-Command", ps_cmd], stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)
        else:
            os.makedirs(chats_dir, exist_ok=True)
            shutil.copy(tmp_path, f"{chats_dir}/{chat_id}.txt")
            subprocess.Popen(["mempalace", "--palace", palace_dir, "mine", chats_dir, "--mode", "convos"], stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)
