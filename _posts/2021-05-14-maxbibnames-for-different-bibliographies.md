---
title: Different maximum number of authors in BibLaTeX bibliography
date: 2021-05-14 13:19
categories: LaTeX
tags: latex
---

I have a document with multiple bibliographies, which I need to submit
somewhere. Very annoyingly, the bibliography counts towards page limit, so I try
to save space whenever possible. One of the places where I save space is that I
limit how many authors are printed for each bibliography entry.

Later it turned out that I need to print full author list at least for the first
bibliography. To minimise the space lost, I decided that I want to set the
number of authors printed for the first bibliography, but not for the second. I
did not find a solution on the Internet, but, fortunately, my colleague (Jan
Bierbaum) is well-versed in LaTeX. He came up with the following solution.

```tex
\documentclass[a4paper]{scrbook}

\usepackage[T1]{fontenc}
\usepackage{lmodern}
\usepackage[utf8]{inputenc}
\usepackage[english]{babel}

\usepackage[%
 bibencoding=utf8,
 backend=biber,
 bibstyle=ieee,
 citestyle=numeric-comp,
 defernumbers=true,
 sortcites=true,
 maxcitenames=2,
 mincitenames=1,
 maxbibnames=3,
 minbibnames=2,
]{biblatex}

\begin{filecontents}{\jobname.bib}
@inproceedings{own,
  title = {Hardware Performance Variation: {{A}} Comparative Study Using Lightweight Kernels},
  author = {Weisbach, Hannes and Gerofi, Balazs and Kocoloski, Brian and Härtig, Hermann and Ishikawa, Yutaka},
  keywords = {OWN}
}

@inproceedings{other,
  title = {Accommodating Thread-Level Heterogeneity in Coupled Parallel Applications},
  author = {Gutiérrez, Samuel K. and Davis, Kei and Arnold, Dorian C. and Baker, Randal S. and Robey, Robert W.
and McCormick, Patrick and Holladay, Daniel and Dahl, Jon A. and Zerr, R. Joe and Weik, Florian and Junghans, Ch
ristoph},
}
\end{filecontents}

\addbibresource{test.bib}


\begin{document}

\section{Own stuff}

Own cite~\cite{own} by \citeauthor{own}
Other cite~\cite{other} by \citeauthor{other}

{
  \expandafter\def\csname blx@maxbibnames\endcsname{99}%
  \printbibliography[heading=none, keyword=OWN]
}


\section{Other stuff}
Own cite~\cite{own} by \citeauthor{own}
Other cite~\cite{other} by \citeauthor{other}
\printbibliography[heading=none, notkeyword=OWN]

\end{document}
```
