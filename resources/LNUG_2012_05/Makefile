slides.pdf: slides.tex *.svg
	pdflatex -shell-escape slides && pdflatex -shell-escape slides && pdflatex -shell-escape slides

#slides.tex: slides.lhs
#	lhs2TeX -o slides.tex --poly slides.lhs

clean:
	rm -f *.aux *.bbl *.blg *.log *.ptb *.toc *.nav *.snm *.out *.vrb

