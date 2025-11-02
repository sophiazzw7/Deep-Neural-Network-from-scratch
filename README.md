/*==============================================================*
 * C) Missing Summary (Character Variables - concise)
 *==============================================================*/

title "Missing Summary (Character Variables)";
proc means data=cleaned_data n nmiss;
  var _character_;
run;
title;
