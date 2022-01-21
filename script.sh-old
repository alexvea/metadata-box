#!/usr/bin/env bash
############################################################
# Help                                                     #
############################################################
Help()
{
   # Display Help
   echo "The script will work on a specific Box folder"
   echo "to convert exif metadata to box metadata with chl-icono template."
   echo "Syntax: scriptTemplate [-h|a|d|r]"
   echo "options:"
   echo "h     Print this Help."
   echo "a     Add metadata"
   echo "d     Delete metadata"
   echo "r     Recursive"
   echo "n     Number of simultaneous process"
   echo
}

############################################################
############################################################
# Main program                                             #
############################################################
############################################################
current_parent_pid=$$
OIFS="$IFS"
IFS=$'\n'
token_refresh_rate_in_sec=300
token_red_period_in_sec=30
scope="xxxxxxxxxxxxxxxxx"
metadata_template="xxxxxxxxxxxxxx"
temp_path="temp"
mkdir $temp_path 2> /dev/null
insert_metadata_logfile=script.log
parent_folder_id="xxxxxxxxxxxxxxxxxxx" #dossier ICONO-BACKUP
#client_id="xxxxxxxxxxxxxxxxxxxxxxxx" #compte avea@sfa.fr
#client_secret="xxxxxxxxxxxxxxxxxxxxxxxxxxxxx" #compte avea@sfa.fr
excluded_file_extension=".xls .m4a .DS_Store .csv .psd .xlsx .BridgeSort .pdf .mp4 .db .html .htm .gif .apdisk"

#template json metadonnée chl icono avec à droite les paramètre exiftool
chlicone_metadata_creation_json=$( cat << EOF
{
    "titre":"Title;Caption-Abstract;XPTitle",
    "objet":"Description;ImageDescription",
    "motsCls":"Subject;Keywords;XPKeywords",
    "auteurs":"Creator;By-line;Artist;XPArtist",
    "priseDeVue":"Create Date;Date Created;Create Date;Date Time Original",
    "copyright":"Rights;CopyrightNotice;Copyright",
    "largeur":"Image Width",
    "hauteur":"Image Height",
    "dateDacquisition":"Date Acquired",
    "rsolutionHorizontale":"X Resolution",
    "rsolutionVerticale":"Y Resolution"
}
EOF
)

clean_up_all(){
        ps aux | grep -ie "set_" | awk '{print $2}' | xargs kill -9
}

request_to_api_box(){
        task=$1
        file_id=$2
        default_sleep_download=0
        do_curl=1
        number_retry=0
        while [ $do_curl -eq 1 ]; do
                sleep $default_sleep_download
                token_status="$(cat token_file | awk -F";" '{print $2}')"
                if [ $task != "REFRESH" ]; then
                        while [ $token_status == "R" ];do
                                sleep 5
                                token_status="$(cat token_file | awk -F";" '{print $2}')"
                        done
                fi
                token=$(cat token_file | awk -F";" '{ print $1 }')
                case $task in
                        "REFRESH")
                                refresh_token=`cat refresh_token_file`
                                http_response_code=$(curl  -w "%{http_code}" -o $temp_path/refresh_token_response -s -i -X POST "https://api.box.com/oauth2/token" \
                                          -H "Content-Type: application/x-www-form-urlencoded" \
                                          -d "client_id=$client_id" \
                                          -d "client_secret=$client_secret" \
                                          -d "refresh_token=$refresh_token" \
                                          -d "grant_type=refresh_token")
                        ;;
                        "LIST")
                                 http_response_code=$(curl -w "%{http_code}" -o $temp_path/$folder_id -s -X GET "https://api.box.com/2.0/folders/$folder_id/items?limit=1000" \
                                                      -H "Authorization: Bearer $token")
                        ;;
                        "CHECK")
                                 http_response=$(curl -w "%{http_code}" -s -X GET "https://api.box.com/2.0/files/$file_id/metadata/$scope/$metadata_template" -H "Authorization: Bearer $token" | jq length )
                                 http_response_code=$(echo ${http_response} | awk '{print $2}')
                        ;;
                        "DOWNLOAD")
                                KB_TO_DOWNLOAD=100
                                B_TO_DOWNLOAD=$((KB_TO_DOWNLOAD*1024))

                                http_response_code=$(curl -w "%{http_code}" -s -o $temp_path/$file_id --header "Range: bytes=0-${B_TO_DOWNLOAD}" -X GET "https://api.box.com/2.0/files/$file_id/content" \
                                -H "Authorization: Bearer $token" \
                                -L)
                        ;;
                        "CREATE")
                                metadata_box_json=$temp_path/$file_id.json
                                 http_response_code=$(curl -w "%{http_code}" -s -i -X POST "https://api.box.com/2.0/files/$file_id/metadata/$scope/$metadata_template" \
                                -H "Authorization: Bearer $token" \
                                -H "Content-Type: application/json" \
                                -d @$metadata_box_json \
                                | head -n 1 | cut -d$' ' -f2)
                        ;;
                        "DELETE")
                                http_response_code=$(curl -w "%{http_code}" -s -i -X DELETE "https://api.box.com/2.0/files/$file_id/metadata/$scope/$metadata_template" \
                                -H "Authorization: Bearer $token" \
                                | head -n 1 | cut -d$' ' -f2)
                        ;;
                esac
                test_http_response $http_response_code $file_id $task $number_retry $token
                default_sleep_download=$(expr $default_sleep_download \* 2 + 2)
                if [ "$http_response_code" -eq 429 ] || [ "$http_response_code" -eq 401 ];then number_retry=$((number_retry+1)); else do_curl=0;fi
        done
        echo $http_response
}

check_metadata_in_box() {
        task="CHECK"
        file_id=$1
        http_response=$(request_to_api_box $task $file_id)
        metadata_check_number=$(echo ${http_response} | awk '{print $1}')
        http_response_code=$(echo ${http_response} | awk '{print $2}')
        #commented because to have the json, you need to download the file once
        #json_keys_number=$(cat $temp_path/$file_id.json | wc -l)
        #metadata_diff=$(expr $metadata_check_number - $json_keys_number)
        case $metadata_check_number in
                8) #cas metadata sur box vide
                         remove_metadata_from_file $file_id &&  download_photo_from_box $file_id &
                ;;
                *) #autres cas
                         download_photo_from_box $file_id &
                #       if [ $metadata_diff -eq 6 ]; then
                #               #cas nb metadata photo = nb metadata box
                #               echo $(date -u) "$file_id has equal metadatas as box metadata.">> $insert_metadata_logfile
                #       elif [ $metadata_diff -gt 6 ]; then
                #               echo $(date -u) "$file_id has less metadatas than box metadata.">> $insert_metadata_logfile
                #       elif [ $metadata_diff -lt 6 ] && [ $metadata_diff -gt 0 ] ; then
                #               echo $(date -u) "$file_id has more metadatas than box metadata.">> $insert_metadata_logfile
                #               remove_metadata_from_file $file_id &&  download_photo_from_box $file_id &
                #       elif [  $metadata_diff -lt 0 ]; then
                #               echo $(date -u) "$file_id has no box metadata chlicono template.">> $insert_metadata_logfile
                #       fi
                ;;
        esac
}

test_http_response(){
        task=$3
        http_code=$1
        file_id=$2
        retry=$4
        used_token=$5
        case $http_code in
                201)
                        echo $(date -u) "code http 201 : metadata $metadata_template created on file during $task$retry $file_id $used_token"  >> $insert_metadata_logfile
                ;;
                204)
                        echo $(date -u) "code http 204 : metadata $metadata_template deleted for file during $task$retry $file_id $used_token" >> $insert_metadata_logfile
                ;;
                206)
                        echo $(date -u) "code http 206 : file HEADER downloaded on box during $task$retry $file_id $used_token" >> $insert_metadata_logfile
                ;;
                404)
                        echo $(date -u) "error http 404 : metadata $metadata_template not existing on file during $task$retry $file_id $used_token" >> $insert_metadata_logfile
                ;;
                401)
                        echo $(date -u) "error http 401 : access forbidden during $task$retry $file_id $used_token">> $insert_metadata_logfile
                ;;
                409)
                        echo $(date -u) "error http 409 : metadata $metadata_template already existing for file during $task$retry $file_id $used_token">> $insert_metadata_logfile
                ;;
                429)
                        echo $(date -u) "error http 429 : Too Many Requests file during $task$retry $file_id $used_token" >> $insert_metadata_logfile
                ;;
                *)
                        echo $(date -u) "code $http_code : ressource $file_id during $task$retry $used_token" >> $insert_metadata_logfile
                ;;
        esac
}

create_metadata_on_file() {
        task="CREATE"
        file_id=$1
        metadata_box_json=$temp_path/$file_id.json
        img=$temp_path/$file_id
        all_exiftool_output=$temp_path/$file_id.exiftool
        exiftool $img > $all_exiftool_output
        if [ $(cat $all_exiftool_output | wc -l) -gt 15 ]  #check if it's an picture
        then
        [[ -z $metadata_box_json ]] && rm $metadata_box_json
        echo $chlicone_metadata_creation_json | jq . > $metadata_box_json
        for key_placeholder in `cat $metadata_box_json | jq -r 'to_entries[] | "\(.key):\(.value)"'`
        do
                key=${key_placeholder%%:*}
                placeholders_alternative=${key_placeholder##*:}
                array_placeholder=($(echo $placeholders_alternative | tr ';' "\n"))
                for placeholder in ${array_placeholder[@]}; do
                        value=$(cat $all_exiftool_output | grep "^${placeholder}\s*:" | awk -F" : " '{print $2}' | head -1)
                        [ -n "$value" ] && break
                done

                case ${key} in
                        "titre" | "objet" | "auteurs" | "copyright" | "motsCls" )
                                value=$(echo "$value" | sed 's/\"/\\"/g' | sed 's/[^[:alnum:]]/\\&/g') # reformatting value without quote, double quote, ampersand
                                 [[ -z $value ]] || value=\"${value}\"
                        ;;
                        "priseDeVue" | "dateDacquisition")
                                value=$(date -d `echo $value | sed 's/:/\//g;s/\//:/3g'` +'%Y-%m-%dT%H:%M:%SZ' 2> /dev/null)
                                [[ -z ${value} ]] || value=\"${value}\"
                        ;;
                        *)
                        ;;
                esac
                [[ -z "$value" ]] && sed -i "/${key}/d" $metadata_box_json || sed -i "s~\"${placeholders_alternative}\"~${value}~g" $metadata_box_json
        done
        sed -i '$ d'  $metadata_box_json
        sed -i '$s/,$//'  $metadata_box_json
        echo "}" >>  $metadata_box_json

                if [[ $(cat $metadata_box_json | wc -l) -eq 2 ]]
                then
                        echo $(date -u) "json $file_id is empty " >> $insert_metadata_logfile
                else
                        http_response=$(request_to_api_box $task $file_id)
                fi
        else
                echo $(date -u) "file $file_id is not an picture " >> $insert_metadata_logfile
        fi
        rm $temp_path/$file_id
}


download_photo_from_box(){
        task="DOWNLOAD"
        file_id=$1
        http_response=$(request_to_api_box $task $file_id)
        create_metadata_on_file $file_id
}

remove_metadata_from_file(){
        task="DELETE"
        file_id=$1
        http_response=$(request_to_api_box $task $file_id)
}

list_items_in_box_folder(){
        task="LIST"
        folder_id=$1
        http_response=$(request_to_api_box $task $folder_id)
        cat $temp_path/$folder_id | jq -r '.entries[] | "\(.type);\(.id);\(.name)"'
}

refresh_token_loop() {
        task="REFRESH"
        while :; do
                http_response=$(request_to_api_box $task)
                refresh_token_generated=$(cat  $temp_path/refresh_token_response | grep -oP "refresh_token\":\"\K[\d\D]*" | cut -d"\"" -f1)
                token_generated=$(cat $temp_path/refresh_token_response| grep -oP ":\"\K[\d\D]*" | cut -d"\"" -f1)
                echo "$refresh_token_generated" > refresh_token_file
                echo "$token_generated;G" > token_file
                echo $(date -u) " refresh token refreshed"  >> $insert_metadata_logfile
                echo $(date -u) " token refreshed"  >> $insert_metadata_logfile
                sleep $(expr $token_refresh_rate_in_sec - $token_red_period_in_sec)
                sed -i "s/;G/;R/g" token_file
                echo $(date -u) " token status RED"  >> $insert_metadata_logfile
                sleep $token_red_period_in_sec
        done
}


############################################################
# Process the input options. Add options as needed.        #
############################################################
# Get the options

num_procs=2
recursive=false
action="no"
while getopts "n:arhd" option; do
   case $option in
      h) # display Help
         Help
         exit;;
      n) # number of simultaneous process
         re='^[0-9]+$'
         if ! [[ $OPTARG =~ $re ]] ; then
                 echo "error: Not a number" >&2
                 exit
         else
                if [ $OPTARG -lt 2 ]; then
                        echo "need to put more then 1 processus"
                else
                        num_procs=$OPTARG
                fi
         fi
         ;;
      a) #add metadata case
         [[ $action == "no" ]] && action="add" || action="error"
         ;;
      d) #delete metadata case
         [[ $action == "no" ]] && action="delete" || action="error"
         ;;
      r)
         recursive=true
         ;;
     \?) # Invalid option
         exit;;
   esac
done

[ $OPTIND -eq 1 ] && Help && exit

trap clean_up_all SIGINT SIGQUIT
echo "number of processus: "$num_procs
echo "action : "$action
echo "recursive: "$recursive

refresh_token_loop &
refresh_token_loop_pid=$!
sleep 5

num_jobs="\j"
scan_box_folder(){
        folder_id=$1
        for item in `list_items_in_box_folder $folder_id`
        do
                while (( ${num_jobs@P} >= num_procs )); do
                    wait -n
                done
                item_type=$(echo ${item} | awk -F";" '{print $1}')
                item_id=$(echo ${item} | awk -F";" '{print $2}')
                item_name=$(echo ${item} | awk -F";" '{print $3}')
                case $item_type in
                        "folder")
                                [[ $recursive == true ]] && scan_box_folder $item_id &
                        ;;
                        "file")

                                filename=$(basename -- "$item_name")
                                extension=".${filename##*.}"
                                [[ $(echo $excluded_file_extension | grep -i $extension) ]] &&  echo $(date -u) "excluded file : $item_id by rules with $extension extension"  >> $insert_metadata_logfile && continue
                                case $action in
                                        "no")
                                                echo "please precise action"
                                        ;;
                                        "error")
                                                echo "you can't add and delete at the same time"
                                        ;;
                                        "add")
                                                check_metadata_in_box $item_id

                                        ;;
                                        "delete")
                                                remove_metadata_from_file $item_id &
                                        ;;
                                esac
                        ;;
                esac
        done
}

echo $(date -u) "######## Début du script pour le dossier $parent_folder_id" >> $insert_metadata_logfile
echo $(date -u) "number of processus: $num_procs " >> $insert_metadata_logfile
echo $(date -u) "action : $action " >> $insert_metadata_logfile
echo $(date -u) "recursive: $recursive " >> $insert_metadata_logfile
echo $current_parent_pid
scan_box_folder $parent_folder_id

while [ $( pgrep -P $current_parent_pid | wc -l ) -ne 2 ]
do
                sleep 6
done
echo $(date -u) "######## Fin du script pour le dossier $parent_folder_id" >> $insert_metadata_logfile

#list_items_in_box_folder $parent_folder_id
#download_photo_from_box 904850776270 && create_metadata_on_file # fichier JPEG
#download_photo_from_box 904850777470 && create_metadata_on_file # fichier CSV

kill $refresh_token_loop_pid

# cas identifiés
# cas fichier non photo OK
# cas fichier PSD OK
# cas photo sans metadata NOK
# cas photo sans resolution (derniere clé json) OK
# cas
