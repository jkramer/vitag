#!/bin/bash
# vitag, Copyright (C) 2007-2009 by Jonas Kramer.
# Published under the terms of the GNU General Public License (GPL).

if [ -z "$EDITOR" ]; then
	echo "\$EDITOR not set." >&2
	exit -1
fi

for PROGRAM in "vorbiscomment" "file" "mktemp" "cat" "cmp" "id3tag" "id3info" "metaflac"
do
	PROGPATH="`which "$PROGRAM" 2>"/dev/null"`"
	if [ -z "$PROGPATH" ]; then
		echo "$PROGRAM is not in your \$PATH." >&2
		exit -1
	fi
	export $PROGRAM="$PROGPATH"
done

function id3_genre ()
{
	GENRE=${1// /.*}

	# Try full match first.
	CODE="`sed -r -n -e "s/GENRE\+\+([0-9]+)\+\+${GENRE}/\1/ip" $0 | head -n 1`"

	# Try partial match
	if [ -z "$CODE" ]; then
		CODE="`sed -r -n -e "s/GENRE\+\+([0-9]+)\+\+.*${GENRE}/\1/ip" $0 | head -n 1`"
	fi

	# Try to match code.
	if [ -z "$CODE" ]; then
		CODE="`sed -r -n -e "s/GENRE\+\+\(?(${GENRE})\)?\+\+.*/\1/ip" $0 | head -n 1`"
	fi

	# No result.
	if [ -z "$CODE" ]; then
		CODE=0
	fi

	echo "$CODE"
}

OVERALL="`mktemp`"

for FILE in "$@"; do
	if [ ! -f "$FILE" ]; then
		echo "$FILE does not exist." >&2
		continue
	fi

	TYPE="`$file "$FILE" | tr '[a-z]' '[A-Z]'`"
	TYPE="${TYPE#$FILE: }"

	# Determine file type (OGG Vorbis and MP3 supported).
	case "$TYPE" in
		*VORBIS*)
			TYPE='OGG'
			;;
		*MPEG*)
			TYPE='MP3'
			;;
		*FLAC*)
			TYPE='FLAC'
			;;
		*)
			echo "$FILE doesn't look like a supported file." >&2
			continue
	esac

	TAGFILE="`mktemp`"

	# Get the tags.
	if [ "$TYPE" = "OGG" ]; then
		if ! $vorbiscomment -l "$FILE" > "$TAGFILE"; then
			echo "Failed to get tags of file \"$FILE\"." >&2
			continue
		fi
	elif [ "$TYPE" = 'FLAC' ]; then
		if ! $metaflac '--export-tags-to=-' "$FILE" > "$TAGFILE"; then
			echo "Failed to get tags of file \"$FILE\"." >&2
			continue
		fi
	elif [ "$TYPE" = "MP3" ]; then
		if ! $id3info "$FILE" | grep '^===' > "$TAGFILE"; then
			echo "Failed to get tags of file \"$FILE\"." >&2
			continue
		fi

		sed -i -r -n \
			-e 's/^=== (TRCK|TPE1|TIT2|TALB|TCON|COMM|TYER) \(.+?\): (.+)$/\1=\2/p' \
			"$TAGFILE"

		# Ugly translation needed.
		sed -i -e 's/^TRCK=/TRACKNUMBER=/' "$TAGFILE"
		sed -i -e 's/^TPE1=/ARTIST=/' "$TAGFILE"
		sed -i -e 's/^TIT2=/TITLE=/' "$TAGFILE"
		sed -i -e 's/^TALB=/ALBUM=/' "$TAGFILE"
		sed -i -e 's/^TCON=/GENRE=/' "$TAGFILE"
		sed -i -e 's/^TYER=/YEAR=/' "$TAGFILE"
	fi

	# Fill up misssing tags.
	for TAG in "TRACKNUMBER" "ARTIST" "TITLE" "ALBUM" "GENRE" "YEAR"; do
		if ! grep -i -q "^$TAG=" "$TAGFILE"; then
			echo "$TAG=" >> "$TAGFILE"
		fi
	done

	# Store tags.
	echo "-- $TYPE -- $FILE" >> "$OVERALL"
	$cat "$TAGFILE" >> "$OVERALL"
	echo "-----" >> "$OVERALL"

	rm "$TAGFILE"
done

if [ "`du -b "$OVERALL" | awk '{ print $1 }'`" = 0 ]; then
	echo "No tags to edit."
	rm $OVERALL
	exit
fi

BACKUP="`mktemp`"
cp -f "$OVERALL" "$BACKUP"

$EDITOR "$OVERALL"

FILE=""
TYPE=""
TAGFILE="`mktemp`"

# Write tags back to audio files from tag file.
if ! $cmp "$OVERALL" "$BACKUP" > "/dev/null"; then
	while read LINE; do

		# Head of tag block.
		if grep -i -q "^-- " <<<"$LINE"; then
			LINE="${LINE#-- }"
			TYPE="${LINE% --*}"
			FILE="${LINE#$TYPE -- }"

		# End of tag block, write tags now.
		elif [ "$LINE" = "-----" ]; then
			if [ "$FILE" != "" ]; then
				if [ "$TYPE" = "OGG" ]; then
					# Do something about "YEAR" here, looks like vorbis doesn't
					# have a field for that one.
					$vorbiscomment -w -c "$TAGFILE" "$FILE"

				elif [ "$TYPE" = 'FLAC' ]; then
					$metaflac '--remove-all-tags' "--import-tags-from=$TAGFILE" "$FILE"

				elif [ "$TYPE" = "MP3" ]; then
					# Ugly...
					sed -i -e 's/^TRACKNUMBER=/track=/' "$TAGFILE"
					sed -i -e 's/^ARTIST=/artist=/' "$TAGFILE"
					sed -i -e 's/^TITLE=/song=/' "$TAGFILE"
					sed -i -e 's/^ALBUM=/album=/' "$TAGFILE"
					sed -i -e 's/^GENRE=/genre=/' "$TAGFILE"
					sed -i -e 's/^YEAR=/year=/' "$TAGFILE"

					while read TAG; do
						KEY="${TAG%=*}"
						VALUE="${TAG#$KEY=}"

						if [ $KEY = 'genre' ]; then
							VALUE=`id3_genre "$VALUE"`
						fi

						$id3tag "--$KEY=$VALUE" "$FILE" >"/dev/null"
					done < "$TAGFILE"
				fi
			fi
			echo -n "" > "$TAGFILE"
		else
			if grep -i -q -E '^(tracknumber|artist|title|album|genre|year)=' <<<"$LINE"; then
				echo "$LINE" >> "$TAGFILE"
			fi
		fi
	done < "$OVERALL"
else
	echo "No tags changed." >&2
fi

rm "$OVERALL"
rm "$TAGFILE"
rm "$BACKUP"

exit

# ID3 genre list for code translation
GENRE++0++Blues
GENRE++1++Classic Rock
GENRE++2++Country
GENRE++3++Dance
GENRE++4++Disco
GENRE++5++Funk
GENRE++6++Grunge
GENRE++7++Hip-Hop
GENRE++8++Jazz
GENRE++9++Metal
GENRE++10++New Age
GENRE++11++Oldies
GENRE++12++Other
GENRE++13++Pop
GENRE++14++R&B
GENRE++15++Rap
GENRE++16++Reggae
GENRE++17++Rock
GENRE++18++Techno
GENRE++19++Industrial
GENRE++20++Alternative
GENRE++21++Ska
GENRE++22++Death Metal
GENRE++23++Pranks
GENRE++24++Soundtrack
GENRE++25++Euro-Techno
GENRE++26++Ambient
GENRE++27++Trip-Hop
GENRE++28++Vocal
GENRE++29++Jazz+Funk
GENRE++30++Fusion
GENRE++31++Trance
GENRE++32++Classical
GENRE++33++Instrumental
GENRE++34++Acid
GENRE++35++House
GENRE++36++Game
GENRE++37++Sound Clip
GENRE++38++Gospel
GENRE++39++Noise
GENRE++40++AlternRock
GENRE++41++Bass
GENRE++42++Soul
GENRE++43++Punk
GENRE++44++Space
GENRE++45++Meditative
GENRE++46++Instrumental Pop
GENRE++47++Instrumental Rock
GENRE++48++Ethnic
GENRE++49++Gothic
GENRE++50++Darkwave
GENRE++51++Techno-Industrial
GENRE++52++Electronic
GENRE++53++Pop-Folk
GENRE++54++Eurodance
GENRE++55++Dream
GENRE++56++Southern Rock
GENRE++57++Comedy
GENRE++58++Cult
GENRE++59++Gangsta
GENRE++60++Top 40
GENRE++61++Christian Rap
GENRE++62++Pop/Funk
GENRE++63++Jungle
GENRE++64++Native American
GENRE++65++Cabaret
GENRE++66++New Wave
GENRE++67++Psychadelic
GENRE++68++Rave
GENRE++69++Showtunes
GENRE++70++Trailer
GENRE++71++Lo-Fi
GENRE++72++Tribal
GENRE++73++Acid Punk
GENRE++74++Acid Jazz
GENRE++75++Polka
GENRE++76++Retro
GENRE++77++Musical
GENRE++78++Rock & Roll
GENRE++79++Hard Rock
GENRE++80++Folk
GENRE++81++Folk-Rock
GENRE++82++National Folk
GENRE++83++Swing
GENRE++84++Fast Fusion
GENRE++85++Bebob
GENRE++86++Latin
GENRE++87++Revival
GENRE++88++Celtic
GENRE++89++Bluegrass
GENRE++90++Avantgarde
GENRE++91++Gothic Rock
GENRE++92++Progressive Rock
GENRE++93++Psychedelic Rock
GENRE++94++Symphonic Rock
GENRE++95++Slow Rock
GENRE++96++Big Band
GENRE++97++Chorus
GENRE++98++Easy Listening
GENRE++99++Acoustic
GENRE++100++Humour
GENRE++101++Speech
GENRE++102++Chanson
GENRE++103++Opera
GENRE++104++Chamber Music
GENRE++105++Sonata
GENRE++106++Symphony
GENRE++107++Booty Bass
GENRE++108++Primus
GENRE++109++Porn Groove
GENRE++110++Satire
GENRE++111++Slow Jam
GENRE++112++Club
GENRE++113++Tango
GENRE++114++Samba
GENRE++115++Folklore
GENRE++116++Ballad
GENRE++117++Power Ballad
GENRE++118++Rhythmic Soul
GENRE++119++Freestyle
GENRE++120++Duet
GENRE++121++Punk Rock
GENRE++122++Drum Solo
GENRE++123++A capella
GENRE++124++Euro-House
GENRE++125++Dance Hall
