Slurm
=====

⦾ **Why does the** ``nvidia-smi`` **command take several seconds to minutes to return output, particularly after a reboot or fresh NVIDIA driver installation?**

**Potential Cause:**

On some GPU compute nodes, the NVIDIA driver may intermittently behave slowly during normal runtime operation. This slow behavior is observed when GPU persistence mode is not enabled, causing delayed responses from GPU management commands such as ``nvidia-smi``.

**Resolution:**:

1. Enable GPU persistence mode so that the NVIDIA driver keeps GPUs initialized even when idle. This prevents repeated GPU reinitialization and ensures nvidia-smi responds immediately.

2. To enable Persistence Mode, run the following command on the GPU node::

        nvidia-smi -pm 1

3. To verify that persistence mode is enabled::

        sudo nvidia-smi -q | grep "Persistence Mode"

Expected output should show ``Persistence Mode`` as ``Enabled``:

.. image:: ../images/troubleshooting_nvidia_persistence_mode.png


⦾ **Why do GPU jobs fail to submit when requesting GPU resources using --gres=gpu:<count>? The submission fails with:

.. image:: ../images/troubleshoot_gpu_gres1.png

**Potential Cause**

This occurs because GPU nodes are missing the Gres= configuration in /etc/slurm/slurm.conf (shows as Gres=null or not present). The Redfish API used to get the GPU count for Slurm nodes is returning GPU count as zero. As a result, slurm.conf is not updated with the GPU count for each Slurm node.

**Resolution**

.. note:: 
        1. Ensure there is network connectivity to all specified BMC IP addresses before executing the script.
        2. Incorrect BMC credentials results in authentication failures. All the BMC credentials must be the same.

1. Login to omnia core container.
2. ssh to Slurm controller.
3. Copy the provided script content and save it with the following filename: ``gpu_detect.sh``
4. Edit the script and update the BMC credentials with the **exact username and password** for your environment. Locate and update the following variables inside the script. ::
        
        BMC_USERNAME=<your_bmc_username>
        BMC_PASSWORD=<your_bmc_password>

 ::

        #!/bin/bash
        set -euo pipefail

        SLURM_CONF="/etc/slurm/slurm.conf"
        LOG="/tmp/gpu_detection.log"

        BMC_USERNAME="****"
        BMC_PASSWORD="****"

        [[ $EUID -eq 0 ]] || { echo "Run as root"; exit 1; }
        for b in curl jq awk sed grep; do command -v "$b" >/dev/null || exit 1; done
        [[ -f "$SLURM_CONF" ]] || { echo "slurm.conf not found"; exit 1; }

        echo "# GPU Detection $(date)" > "$LOG"

        # ---------------- GPU DETECTION ----------------
        detect_gpus() {
                local ip="$1"

                curl -ksu "$BMC_USERNAME:$BMC_PASSWORD" \
                        "https://$ip/redfish/v1/" >/dev/null || { echo 0; return; }

                curl -ksu "$BMC_USERNAME:$BMC_PASSWORD" \
                        "https://$ip/redfish/v1/Chassis/System.Embedded.1/PCIeDevices" |
                jq -r '.Members[]?.["@odata.id"]' |
                while read -r dev; do
                        curl -ksu "$BMC_USERNAME:$BMC_PASSWORD" "https://$ip$dev"
                done |
                jq -r '
                select(
                        (
                                (.ClassCode=="0x0300" or .ClassCode=="0x0302") and
                                (.VendorId=="0x10de" or .VendorId=="0x1002")
                        )
                        or
                        (
                                (.Manufacturer // "" | test("NVIDIA|AMD"; "i")) and
                                (.Name // "" | test("GPU|RTX|TESLA|A100|H100|L40|GB"; "i"))
                        )
                ) | .Name
                ' | wc -l
        }

        # ---------------- NODE UPDATE ----------------
        update_node() {
                local node="$1" gpus="$2"

                local match
                match=$(grep -n "^NodeName=.*\b$node\b" "$SLURM_CONF" || true)
                [[ -z "$match" || "$match" =~ \[ ]] && return 1

                local ln=${match%%:*}
                local orig
                orig=$(sed -n "${ln}p" "$SLURM_CONF")

                local new
                new=$(echo "$orig" | sed -E 's/[[:space:]]+Gres=[^[:space:]]+//g')
                new="$new Gres=gpu:$gpus"

                awk -v l="$ln" -v r="$new" 'NR==l{$0=r}1' \
                        "$SLURM_CONF" > "$SLURM_CONF.tmp"
                mv "$SLURM_CONF.tmp" "$SLURM_CONF"
        }

        # ---------------- GLOBAL CONFIG ----------------
        ensure_globals() {

        # Ensure SelectType
        if grep -q "^SelectType=" "$SLURM_CONF"; then
                sed -i 's/^SelectType=.*/SelectType=select\/cons_tres/' "$SLURM_CONF"
        else
                echo "SelectType=select/cons_tres" >> "$SLURM_CONF"
        fi

        # Ensure GresTypes near SelectType
        if grep -q "^GresTypes=" "$SLURM_CONF"; then
                if ! grep -q "^GresTypes=.*\bgpu\b" "$SLURM_CONF"; then
                sed -i 's/^GresTypes=\(.*\)/GresTypes=\1,gpu/' "$SLURM_CONF"
                fi
        else
                awk '
                {
                print
                if ($0 ~ /^SelectType=select\/cons_tres/) {
                        print "GresTypes=gpu"
                }
                }' "$SLURM_CONF" > "$SLURM_CONF.tmp" && mv "$SLURM_CONF.tmp" "$SLURM_CONF"
        fi
        }

        # ---------------- MAIN ----------------
        updated=0

        for arg in "$@"; do
                [[ "$arg" =~ : ]] || continue
                ip="${arg%:*}"
                node="${arg#*:}"

                gpus=$(detect_gpus "$ip")
                [[ "$gpus" -eq 0 ]] && continue

                echo "$(date): $node ($ip) -> $gpus GPUs" >> "$LOG"

                if update_node "$node" "$gpus"; then
                        updated=1
                fi
        done

        if [[ $updated -eq 1 ]]; then
                ensure_globals
                echo "✔ slurm.conf updated correctly"
                echo "✔ GresTypes placed after SelectType"
                echo "✔ NO reload applied"
        else
                echo "No GPUs detected or no nodes updated"
        fi

        echo "Log: $LOG" 

5. Run the following command to make the script executable: ::
        
        chmod +x gpu_detect.sh

6. To run the script on a **single node**, use the following format: ::
        
        ./gpu_detect.sh <bmc_ip>:<node_name>
 
 Example::
 
        ./gpu_detect.sh 192.168.1.10:node0

7. To run the script on **multiple nodes**, specify multiple ``<ip>:<node_name>`` pairs separated by spaces ::
        
        ./gpu_detect.sh <bmc_ip1>:<node_name1> <bmc_ip2>:<node_name2>
 
 Example::
 
        ./gpu_detect.sh 192.168.1.10:node01 192.168.1.11:node02
 
8. To verify the GPU count, run the following command on the Slurm controller. If any node is in the INVALID or DRAIN state, the following commands must be executed on the Slurm controller node by changing the ``node_name``:  ::
        
        scontrol update NodeName=<node_name> State=DOWN Reason="stuck completing"
        scontrol reconfigure
        scontrol update NodeName=<node_name> State=RESUME