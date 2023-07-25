slides-for-meetecho.pdf: slides.pdf
	# Workaround for meetecho doing slides at fixed DPI rather than fixed
	# size in pixels
	#
	# Thanks Kurt Pfeifle at https://stackoverflow.com/a/7507511
	gs \
	 -o $@ \
	 -sDEVICE=pdfwrite \
	 -g16000x9000 \
	 -dFIXEDMEDIA \
	 -dPDFFitPage \
	 -dCompatibilityLevel=1.4 \
	  $<

slides.pdf: slides.tex pairing-old.png
	lualatex $<

pairing-old.png: /home/chrysn/git/coap-gatt-demo/screenshots/pairing.png
	convert $<  -fx '((r+g+b)/3) ^ 0.5' $@
