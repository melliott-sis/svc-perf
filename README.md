====== Updates ======
I have updated the python scripts to be compatible with the newest version of the Zabbix protocol. This requires the python protobix library to be installed. I have also found that it may be necessary to install urllib3 via package manager instead of pip. If installed via pip, it may not be properly linked with openssl and this will cause the scripts to fail. For the most part I have disabled checks for valid SSL certificates on the SVC.

svc_mon2.py fails on my test SVC for unknown reasons. I have not been able to find documentation to troubleshoot the API calls against the SVC.

Todo:
- Add a command line/config file argument to enable/disable valid SSL certificate checks
- Fix svc_mon2.py
- Convert scripts to use protobix for updates instead of the external zabbix_sender file

====== Installation ======
- Install Python modules pywbem and pyzabbix:
  easy_install pywbem pyzabbix zbxsend simplejson protobix
    or
  pip install pywbem pyzabbix zbxsend simplejson protobix

- Copy files to appropriate locations, chmod +x shell scripts:
/etc/zabbix/externalscripts: (zabbix scripts)
 svc_mon.py
 svc_mon (chmod +x)
 svc_mon2.py
 svc_mon2 (chmod +x)
 svc_perf (chmod +x)
 svc_perf.conf (Storwize/Zabbix server authentication configuration)
 svc_perf_discovery_sender (chmod +x)
 svc_perf_discovery_sender.py
 svc_perf_graph.py
 svc_perf_graph (chmod +x)
 svc_perf_wbem.py
 svc_status.awk
 svc_status.sh (chmod +x)

/etc/cron.d:
 svc_perf_cron (Storwize name and schedule configuration)

/var/cache/zabbix: (script-generated cache files and logs)
 svc_mon.errlog
 svc_perf.XXX.cache (cache files must be persistent)
 svc_perf.XXX.errlog
 svc_perf_graph.log

====== Configuration guide ======
svc_* scripts connect to configured Storwize cluster(s) with single login/password specified in the /etc/zabbix/externalscripts/svc_perf.conf ($SVC_USER and $SVC_PWD).
That account must have the Storwize Administrator role (it seems like account with Monitor role can't refresh performance stats).

svc_perf_graph uses Zabbix API to create storage pool aggregate graphs in Zabbix server. Zabbix connection settings is specified in /etc/zabbix/externalscripts/svc_perf.conf ($ZABBIX_SERVER, $ZABBIX_USER, $ZABBIX_PASSWORD)
$ZABBIX_SERVER specifies Zabbix API URL (http://zabbix.domain.com), script connects to "$ZABBIX_SERVER/api_jsonrpc.php". Zabbix account $ZABBIX_USER must have read-write permissions to Storwize nodes in Zabbix.

Storwize cluster name(s) is specified in the /etc/cron.d/svc_perf_cron file. Use short unqualified DNS name paying attention to match Zabbix node name with cluster name.
Note for Storwize V7000 Unified customers: configure scripts to connect to a corresponding Storwize V7000 block device cluster management address!
All svc_* scripts is called from /etc/cron.d/svc_perf_cron file. svc_perf script is called for each cluster, specified job repeat interval (*/3) must match Storwize perfstats refresh interval ("startstats -interval 3").

Storwize configuration:
- Open Storwize GUI, navigate to "Access - Users" and create account named "zabbix", assign it to "Administrator" role ("Monitor" role will not work for unknown reason, sorry!)
Storwize V7000 Unified customers should use their Storwize V7000 block device management GUI!
- Open SSH connection to Storwize and run command to change Storwize stats refresh interval to a practical value as the default (15 minutes) is too long
 startstats -interval 3
- Check /etc/cron.d/svc_perf_cron to match svc_perf script repeat interval with Storwize stats interval
- Configure key-based SSH access for zabbix user to Storwize CLI:
 sudo -u zabbix ssh-keygen
 sudo -u zabbix ssh-keygen -e
 save ssh2 public key to file
- Assign ssh2 public key to Storwize "zabbix" account
Storwize GUI - Access - Users - zabbix - Properties - SSH Public Key - Browse to public key file
- Check svc_status.sh script:
 sudo -u zabbix /etc/zabbix/externalscripts/svc_status.sh <storwize>

Zabbix configuration:
- Manually create value maps from the file zabbix_value_maps.txt
- Import Zabbix templates from files: _Special_Storwize_Perf.xml, _Special_Storwize_Block_Status.xml, _Special_Storwize_Unified_Status.xml
- Create node for Storwize cluster, set node name to short unqualified DNS name, link template _Special_Storwize_Perf
- Create Zabbix account "svc_perf" with read-write permissions to new Storwize nodes

====== Usage ======
- Wait 15 min for cron to run svc_perf_discovery_sender script, check /var/cache/zabbix/svc_perf_discovery_sender log for errors
- Check Storwize nodes in Zabbix GUI: volume and mdisk items will be created
- After 6 min (two svc_perf job repeat intervals) volume/mdisk performance stats will appear in Latest Data tab (check /var/cache/zabbix/svc_perf.XXX.errlog for errors)
- After 15 min aggregate storage pool graphs will appear on Graphs tab
- Every 3 min cron will run svc_perf script and feed fresh stats to Zabbix
- Every 10 min svc_mon script will check Storwize for mdisk status and storage pool space usage
- Every 3 min Zabbix will call svc_status.sh external check for Storwize alerts
